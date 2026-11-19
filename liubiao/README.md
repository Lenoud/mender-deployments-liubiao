# mender-deployments-liubiao

自维护的 Mender `deployments` 服务（部署与 OTA 下发），基于上游 [mendersoftware/mender-server](https://github.com/mendersoftware/mender-server) v4.1.4。只做一件事：仿照 Mender 企业版提供**计划部署**（单个开始时间），供自维护前端 [mender-gui-liubiao](https://git.skyrisai.com/yuzus/mender-gui-liubiao) 的“已计划”页和部署向导使用。

## 仓库约定

| 项目 | 内容 |
|---|---|
| 基线 | tag `upstream-v4.1.4` = 上游 `v4.1.4`（`89f60e8f1d9f2923e66f45d3bc8d21deb5fed475`），取自快照 fork [Lenoud/mender-server](https://github.com/Lenoud/mender-server)，保留上游完整历史 |
| 改动方式 | 除 `liubiao/` 目录外，仓库内容与上游 tag 完全一致。**不直接修改上游源码**，修改写成 `liubiao/patches/*.patch`，构建时应用 |
| 代码托管 | GitLab [yuzus/mender-deployments-liubiao](https://git.skyrisai.com/yuzus/mender-deployments-liubiao)（私有，`main` 受保护，改动走 MR）为主；GitHub [Lenoud/mender-deployments-liubiao](https://github.com/Lenoud/mender-deployments-liubiao)（公开）为快照，GitLab 合入后推送同步 |
| 构建 | GitLab CI，配置在 `liubiao/.gitlab-ci.yml`（项目设置的 CI 配置路径；根目录 `.gitlab-ci.yml` 是上游的，不使用）：`liubiao/verify.sh` 核对基线与补丁 SHA256 → 临时 MongoDB 8.0.28 跑 deployments 全部存储测试 → `liubiao/Dockerfile` 应用补丁、跑公共包与本服务单测、构建镜像。MR 只构建，`main` 推送 Harbor |
| 镜像 | `harbor.skyrisai.com/develop/mender-deployments-liubiao:upstream-4.1.4-main-<commit前8位>-<时间>`，二进制版本号 `v4.1.4+liubiao.<commit前8位>` |
| 部署 | `k8s_services` 的 `mender/values-prod.yaml` 中 `deployments.image` |

`git diff upstream-v4.1.4` 只应看到 `liubiao/` 目录（以及 GitLab 初始化生成的 `.gitlab/merge_request_templates/`）；CI 用 `liubiao/verify.sh` 核对，基线写在 `liubiao/upstream.env`。

## 补丁

补丁按文件名顺序应用，SHA256 登记在 `liubiao/patches/SHA256SUMS`。

| 补丁 | 内容 |
|---|---|
| `0001-deployments-scheduled-deployments.patch` | 计划部署（单阶段定时开始），见下文 |

### 计划部署的行为（与企业版一致）

企业版的计划部署不是独立组件，而是闭源 `deployments` 镜像整体替换开源同名服务后提供的功能：部署在创建时写入数据库，设备取更新时如果它最早的未完成部署还没到开始时间，就什么都不返回；到点后由设备下一次轮询取走，没有定时器。本补丁按同样逻辑实现：

- **创建**：`POST /api/management/v1/deployments/deployments`（以及按分组创建）接受企业版格式的 `phases`，只允许一个阶段、覆盖全部设备（`batch_size` 不填或 100），开始时间最多在 `DEPLOYMENTS_SCHEDULE_MAX_AHEAD`（默认 `72h`，Go duration 格式）之后，否则返回 400。开始时间早于当前时间等同于立即开始。
- **下发**：设备取更新时，最早的未完成部署未到开始时间则返回“暂无更新”，不建设备部署记录，**也不跳过它去取更新的部署**。开源版按创建时间往后找部署，跳过的话设备装完新部署后就再也找不到这个计划部署。
- **列表**：`status=scheduled` 只列未到开始时间、未结束的部署；`status=pending` 排除它们。数据库中计划部署的状态仍是 `pending`，接口输出时显示为 `scheduled`。
- **取消**：沿用原有中止接口 `PUT .../deployments/{id}/status`（`aborted`）。
- **不做**：多阶段分批发布、`uniform_phases`、动态分组（这些在企业版也属于专业版/企业版功能）。

**已知影响**：未开始的计划部署会让同一批设备上之后创建的部署排队，直到它开始执行。前端在创建部署时会提醒并提供取消入口；紧急发布要先取消计划部署。上限越短，影响窗口越短。

## 本地验证补丁

```bash
sh liubiao/verify.sh
git apply liubiao/patches/*.patch
(cd backend && go test -tags nopkcs11 -short ./services/deployments/...)
# 存储测试需要 MongoDB：TEST_MONGO_URL=mongodb://<地址>:27017 go test -tags nopkcs11 ./services/deployments/store/mongo/...
git checkout -- backend && git clean -n backend   # 确认后删除补丁新增的文件
```

## 升级上游

1. 先同步 fork `Lenoud/mender-server` 拿到新 tag，再把新 tag 合入本仓库 `main`（保留上游历史，`liubiao/` 不会冲突），新建 `upstream-<tag>` 基线 tag。
2. `git apply --check` 确认补丁仍能应用；不能应用时按新代码重写补丁并重新跑单测和存储测试，重点核对设备取部署的流程（`getNewDeploymentForDevice`）和按状态查询。
3. 更新 `liubiao/upstream.env`、`liubiao/patches/SHA256SUMS`、Dockerfile 与 CI 中的 Go/Mongo 镜像版本，走 MR 合入后由 CI 构建新镜像。
