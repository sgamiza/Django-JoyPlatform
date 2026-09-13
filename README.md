# Joy QA Platform / 接口测试平台

[English](#english) | [中文](#中文)

Django web platform for HTTP API testing, scheduled execution, Locust load tests, and result reports. The UI title is **接口测试平台**. Source lives under `Joy_QA_Platform/`.

---

## English

### What it is

Teams keep API cases in the browser (project → module → case), pick a host environment, and run them with a vendored [HttpRunner](https://github.com/HttpRunner/HttpRunner) 1.4.7. Celery runs one-shot and looping jobs; Locust can reuse the same YAML as a master/slave load test. MySQL stores platform data; Redis is the cache; RabbitMQ (AMQP) is the Celery broker in `settings.py`.

This tree is a working snapshot (Django 2.1 era). Fill `Joy_QA_Platform/configs.py` before you start. Do not commit real passwords.

### Features

- **Accounts**: register, login, logout, email captcha, password reset (`frame`). Custom user model `frame.UserInfo`. Optional email suffix lock (`EMAIL_SUFFIX`).
- **Home**: charts of tasks per project and failed tasks; recent failure list with date range (`/api/index/`).
- **Projects**: create / list / search / update / soft-delete; owner, testers, developers, publish app. Object permissions via django-guardian (`view` / `update` / `delete`).
- **debugtalk.py**: per-project Python hooks edited in the UI (`/api/debugtalk_list/`).
- **Modules**: belong to a project; create / list / search / update / soft-delete.
- **Cases**: HttpRunner-style fields — URL, method, headers, body, variables, parameters, hooks, extract, validate. Create, list, search, edit, clone, upload, run against a chosen env. Last run env is remembered. Soft-delete.
- **Configs** (sidebar hidden, URLs still there): reusable request templates under a project/module.
- **Environments**: name + `host_port` per project (`/api/env_list`).
- **Tasks**: pick env / project / module / cases, receiver email, start time, optional loop interval. Run, stop, list, monitor. Failures go to `task_failed_record`. Successful loop runs can skip writing a report.
- **Reports**: HtmlTestResult-style summaries stored in MySQL (`report_info`); list / search / query / delete.
- **Locust**: start master from a generated YAML (`locust -f … --master`, bind port from `LOCUST_MASTER_BIND_PORT`, default 8095). Slaves live in `slave/QAPlatform/` (`locustfile.py`, `locust.yml`, `debugtalk.py`).
- **Auth (superuser only)**: assign project permissions (`/api/auth`).
- **Ops around the site**: session timeout 30 minutes; SMTP for captcha/alerts (Tencent Exmail host in settings); file logs under `./logs/`.

### Stack

| Piece | Version / note |
|---|---|
| Python | 3.x compatible with Django 2.1 (typically 3.6–3.7) |
| Django | 2.1.3 |
| HttpRunner | **1.4.7, vendored** in `Joy_QA_Platform/httprunner/` (not in `requirements.txt`) |
| Celery | 4.2.0 + django-celery 3.2.2, django-celery-beat, django-celery-results |
| Flower | 0.9.2 |
| Locust | locustio 0.9.0 |
| MySQL | mysqlclient 1.3.13 |
| Redis | django-redis 4.10.0 |
| Broker | AMQP (`BROKER_URL` in `settings.py`; debug default `amqp://guest:guest@127.0.0.1:5672//`) |
| Permissions | django-guardian 1.4.9 |
| Front-end | Layui, Bootstrap-style frame templates, ECharts on the home page |

Other pins: `requests`, `PyYAML`, `jinja2`, `colorlog`, `flower`, `six`. CSRF middleware is commented out in this snapshot.

### Configure

Edit `Joy_QA_Platform/Joy_QA_Platform/configs.py`:

| Variable | Purpose |
|---|---|
| `DEBUG_DATABASES_*` / `DATABASES_*` | MySQL name, user, password, host, port (3306). `DEBUG` in `settings.py` picks which block. |
| `EMAIL_HOST_USER` / `EMAIL_HOST_PASSWORD` / `EMAIL_FROM` | Captcha and alert mail. SMTP: `smtp.exmail.qq.com:465` SSL. |
| `REDIS_LOCATION` / `REDIS_PASSWORD` | Cache, default `redis://127.0.0.1:6379`. |
| `LOCUST_WORKSPACE_DIR` / `LOCUST_MASTER_BIND_PORT` | Locust work dir name (`QAPlatform`) and master port (`8095`). |
| `IS_CREATE_SUPERUSER` / `SUPERUSER_NAME` / `SUPERUSER_PWD` | Seed admin if the bootstrap path is used. |
| `EMAIL_SUFFIX` | Optional register-email domain filter. |

`SECRET_KEY` and production `BROKER_URL` sit in `Joy_QA_Platform/settings.py`. Point the broker at your own RabbitMQ. `ApiManager/tasks.py` also constructs a Redis Celery app (`redis://127.0.0.1:6379/0` and `/1`); keep Redis up if you use that path.

Create the MySQL schema first, then migrate.

### Run

Need MySQL, Redis, and RabbitMQ (debug guest/guest on localhost is the default broker).

```bash
cd Joy_QA_Platform
python -m venv .venv
# Windows: .venv\Scripts\activate
source .venv/bin/activate
pip install -r requirements.txt

# fill configs.py, then:
python manage.py migrate
python manage.py runserver 0.0.0.0:8000
```

Open `http://127.0.0.1:8000/` — empty path goes to login (`frame/login`). After login, API pages are under `/api/…`.

Celery worker (django-celery):

```bash
python manage.py celery worker -l info
```

Optional Flower for the queue UI. Locust master/slave is started from the platform (`locust_run` / `locust_stop`), not from `runserver` alone.

Create `Joy_QA_Platform/logs/` if file handlers fail on first boot.

### Layout

```
Django-JoyPlatform/
├── LICENSE                          # Apache-2.0
├── README.md
└── Joy_QA_Platform/
    ├── manage.py
    ├── requirements.txt
    ├── Joy_QA_Platform/             # Django project
    │   ├── settings.py
    │   ├── configs.py               # local DB / Redis / mail / Locust
    │   ├── urls.py                  # /admin/ + app/function[/id] dispatcher
    │   ├── activator.py             # maps api → ApiManager, frame → frame
    │   └── wsgi.py
    ├── frame/                       # users, login, captcha
    ├── ApiManager/                  # projects, cases, tasks, reports, locust
    │   ├── models.py
    │   ├── views.py
    │   ├── tasks.py                 # HttpRunner + Celery
    │   └── operations/              # one module per UI area
    ├── httprunner/                  # vendored HttpRunner 1.4.7
    ├── templates/frame/             # login, home chrome, help
    ├── templates/api/               # project / case / task / report pages
    ├── templates/report/            # HTML report samples
    ├── static/                      # layui, frame assets
    ├── suite/                       # sample debugtalk / library
    └── slave/QAPlatform/            # Locust slave templates
```

Routing is reflective: `/(app)/(function)/` and `/(app)/(function)/(id)/`. `app=api` loads `ApiManager.views`; `app=frame` loads `frame.views`. Unauthenticated GET to `/api/…` is sent to the login page.

### License

Apache License 2.0. HttpRunner inside `httprunner/` is MIT (debugtalk). Locust/Celery/Django keep their own licenses.

---

## 中文

### 这是什么

浏览器里按 **项目 → 模块 → 用例** 维护 HTTP 接口用例，选运行环境后，用仓库内嵌的 [HttpRunner](https://github.com/HttpRunner/HttpRunner) 1.4.7 执行。Celery 跑即时任务和循环监控；同一套 YAML 也可以交给 Locust 做主从压测。业务数据在 MySQL，缓存走 Redis，`settings.py` 里 Celery 的 broker 是 RabbitMQ（AMQP）。

界面名称是 **接口测试平台**。这是 Django 2.1 年代的一份可运行快照。启动前先填 `Joy_QA_Platform/configs.py`，不要把真实密码提交进 Git。

### 功能

- **账号**：注册、登录、退出、邮箱验证码、重置密码（`frame`）。用户模型是 `frame.UserInfo`。可用 `EMAIL_SUFFIX` 限制注册邮箱后缀。
- **首页**：任务按项目分布、失败任务趋势、失败列表和日期筛选（`/api/index/`）。
- **项目管理**：增删改查（软删）、负责人/测试/开发/发布应用。django-guardian 对象权限（查看/修改/删除）。
- **驱动 debugtalk.py**：每个项目一份，在「驱动列表」里编辑。
- **模块管理**：挂在项目下，增删改查（软删）。
- **用例管理**：HttpRunner 字段（URL、方法、Header、Body、变量、参数、hooks、extract、validate）。新增、列表、搜索、编辑、克隆、上传、选环境执行。记住上次环境。软删。
- **配置管理**：侧栏默认隐藏，路由仍在；项目/模块下的可复用请求模板。
- **环境管理**：环境名 + `host_port`，按项目隔离。
- **任务管理**：环境/项目/模块/用例、收件邮箱、开始时间、可选循环间隔。运行、停止、列表、监控。失败写入 `task_failed_record`。循环任务全绿时可以不落报告。
- **报告管理**：执行摘要进 MySQL（`report_info`），列表/查询/删除。
- **Locust 压测**：由 YAML 起 master（`locust -f … --master`，端口 `LOCUST_MASTER_BIND_PORT`，默认 8095）。从机模板在 `slave/QAPlatform/`。
- **权限管理**（仅超管）：给项目授权（`/api/auth`）。
- **其它**：会话 30 分钟过期；验证码/告警走 SMTP（settings 里是腾讯企业邮）；日志目录 `./logs/`。

### 技术栈

| 组件 | 版本 / 说明 |
|---|---|
| Python | 需能跑 Django 2.1（常见 3.6–3.7） |
| Django | 2.1.3 |
| HttpRunner | **1.4.7，内嵌** `Joy_QA_Platform/httprunner/`（不在 pip 列表里） |
| Celery | 4.2.0 + django-celery 3.2.2、beat、results |
| Flower | 0.9.2 |
| Locust | locustio 0.9.0 |
| MySQL | mysqlclient 1.3.13 |
| Redis | django-redis 4.10.0 |
| Broker | AMQP（`settings.py` 的 `BROKER_URL`；调试默认本机 guest） |
| 权限 | django-guardian 1.4.9 |
| 前端 | Layui、frame 模板、首页 ECharts |

其它钉死版本见 `requirements.txt`。这份代码里 CSRF 中间件是注释掉的。

### 配置

改 `Joy_QA_Platform/Joy_QA_Platform/configs.py`：

| 变量 | 用途 |
|---|---|
| `DEBUG_DATABASES_*` / `DATABASES_*` | MySQL 库名、账号、密码、主机、端口。由 `settings.py` 的 `DEBUG` 二选一。 |
| `EMAIL_*` | 验证码和告警邮件。SMTP：`smtp.exmail.qq.com:465` SSL。 |
| `REDIS_LOCATION` / `REDIS_PASSWORD` | 缓存，默认本机 6379。 |
| `LOCUST_*` | 压测工作目录名和 master 端口。 |
| `IS_CREATE_SUPERUSER` / `SUPERUSER_*` | 是否按配置创建初始超管。 |
| `EMAIL_SUFFIX` | 可选，限制注册邮箱域名。 |

`SECRET_KEY` 和生产 `BROKER_URL` 在 `settings.py`。请改成自己的 RabbitMQ。`ApiManager/tasks.py` 里还有一套 Redis Celery（`/0`、`/1`），走那条路径时 Redis 必须在。

先建好 MySQL 库再 `migrate`。

### 怎么跑

本机需要 MySQL、Redis、RabbitMQ（调试默认 `127.0.0.1:5672` guest）。

```bash
cd Joy_QA_Platform
python -m venv .venv
# Windows: .venv\Scripts\activate
source .venv/bin/activate
pip install -r requirements.txt

# 填好 configs.py 之后：
python manage.py migrate
python manage.py runserver 0.0.0.0:8000
```

打开 `http://127.0.0.1:8000/`，根路径进登录页。登录后业务页在 `/api/…`。

Celery：

```bash
python manage.py celery worker -l info
```

队列监控可用 Flower。Locust 主从由平台接口拉起，不是只开 `runserver` 就会压测。

若文件日志报错，先建 `Joy_QA_Platform/logs/`。

### 目录

见上文 English 一节的 Layout。路由是反射的：`/(应用)/(函数)/`。`api` 进 `ApiManager.views`，`frame` 进 `frame.views`。未登录访问 `/api/…` 会回到登录页。

### 许可证

Apache-2.0。内嵌 HttpRunner 为 MIT。
