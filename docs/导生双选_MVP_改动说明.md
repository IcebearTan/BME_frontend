# 导生双选 MVP 改动说明

> 本文用于 Review 本次“导生双选”功能。以下仅描述当前本地已实现的内容。

## 1. 功能概述

“导生双选”用于在训练营营期内完成学员与导生的匹配：学员浏览已发布的导生资料，选择一位导生并留下留言；导生维护展示资料并查看选择自己的同学；管理员配置活动并查看最终匹配关系。

功能使用既有用户、营期和成员体系，不替换原营期业务。

## 2. 分支与仓库

本次功能各仓库均在独立分支开发，未直接修改 `develop`：

`codex/feat-mentor-market-mvp`

| 仓库 | 本地提交 | 变更内容 |
| --- | --- | --- |
| `BME_frontend` | `1127d90` | 学员端、导生端页面与入口 |
| `BME_backend` | `913b0a7` | 管理员营期管理面板 |
| `BME_platform_flask` | `10b55eb` | Flask 接口、模型、数据库迁移与本地初始化 |

## 3. 学员端与导生端（BME_frontend）

### 修改文件

- `src/router.js`
- `src/components/Home/StudyHub.vue`
- `src/views/MentorMarketView.vue`

### 已实现能力

- 在学习中心显示“导生双选”入口；入口由当前活动状态控制。
- 学员可浏览已发布导生的海报、年级、方向、简介、已选人数与剩余名额。
- 学员可选择导生、填写留言，并在活动开放期间更换导生。
- 导生使用同一页面维护个人展示资料：上传海报、选择年级与细分方向、填写简介、设置可带人数（3–8 人）。
- 导生可查看已选择自己的学生和学生留言。
- 页面使用现有 Dew UI 组件和设计 Token，支持明暗主题。

### 当前边界

- 学生不在本功能中上传个人资料。
- 学生向导生留言已实现。
- “点击头像进入对方公开主页”尚未实现，不能作为本次已交付能力。

## 4. 管理员端（BME_backend）

### 修改文件

- `src/components/CampSessionDetail.vue`
- `src/components/MentorMarketPanel.vue`

### 已实现能力

- 在营期详情增加“导生拼团”标签页。
- 管理员可控制本营期导生双选活动的开放、关闭等状态。
- 管理员可查看导生展示资料。
- 管理员可查看按“导生 → 学生”分组的最终匹配名单；活动关闭后仍可查询。

管理员接口沿用现有 `teacher`、`super_admin` 权限控制。

## 5. Flask 服务端（BME_platform_flask）

### 修改文件

- `app.py`
- `blueprints/__init__.py`
- `blueprints/mentor_market.py`
- `models.py`
- `config.py`
- `scripts/migrate/migrate_06_mentor_market.py`
- `scripts/migrate/README.md`
- `.env_example`
- `scripts/seed_local.py`

### 数据模型

新增以下模型，不修改原有用户表和营期表：

- `MentorMarketActivity`：营期内活动配置与开放状态。
- `MentorMarketProfile`：导生海报、年级、组别、简介、容量与发布状态。
- `MentorMarketChoice`：学生在指定活动中选择导生的记录与留言。

### 关键业务行为

- 选择导生时使用数据库事务与行锁检查容量，防止超额选择。
- 学员更换导生时更新当前活动的选择记录。
- 选择成功后同步当前营期的 `CampMember.team_mentor_id`，使成员页“归属导生”与双选结果一致。
- 容量可配置为 3–8 人，且不能小于已选学生数。

### 主要接口

| 接口 | 用途 |
| --- | --- |
| `GET /mentor-market/featured` | 获取当前展示活动 |
| `GET /mentor-market/activities/<id>/profiles` | 获取导生列表 |
| `GET/PUT /mentor-market/activities/<id>/my-profile` | 导生读取/保存个人资料 |
| `POST /mentor-market/activities/<id>/my-profile/poster` | 上传导生海报 |
| `GET /mentor-market/activities/<id>/my-choice` | 获取学员当前选择 |
| `POST /mentor-market/activities/<id>/choices` | 学员选择或更换导生 |
| `GET /mentor-market/activities/<id>/my-students` | 导生查看已选学生 |
| `GET /mentor-market/admin/camp-sessions/<camp_session_id>/results` | 管理员查看最终匹配名单 |

## 6. 数据库迁移

新增迁移：

`BME_platform_flask/scripts/migrate/migrate_06_mentor_market.py`

用于创建导生双选的表结构。它应在独立本地开发数据库执行；生产升级需由维护者按生产迁移策略确认后执行。

本地 UI 演示用的 seed / cleanup 脚本不纳入正式提交，不读取或复制真实用户数据。

## 7. 本地联调结果

已完成本地 Flask、MySQL、Redis 环境的联调验证：

1. 学员可浏览多位导生、选择/更换导生，并获取实时名额。
2. 导生可保存资料、调整人数、查看学生与留言。
3. 管理员可查看活动、导生资料与最终匹配名单。
4. 选择结果会同步营期成员的 `team_mentor_id`。
5. 学员端生产构建通过，服务端接口可正常返回。

## 8. 合并注意事项

- 三个仓库应分别使用功能分支发起 Review / 合并请求。
- `.env`、本地数据库、Redis 数据和本地测试海报不得提交。
- `BME_frontend/package-lock.json` 当前存在与本功能无关的版本号改动，已明确不纳入本次提交。
