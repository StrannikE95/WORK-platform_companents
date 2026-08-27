
Ресурсы ([требования](https://docs.gitlab.com/install/requirements/), [поддерживаемые ОС](https://docs.gitlab.com/install/package/)):
+ Учебный self-managed стенд: одна **Linux x86_64** VM, **Ubuntu 24.04**; отдельного продукта «только CI» нет — нужны координатор GitLab и Runner.
+ ПО: **GitLab EE 19.3.0-ee.0** (без лицензии — Free) и **GitLab Runner 19.3.0**; встроенные PostgreSQL/Redis ставятся Linux package. Docker нужен только для Docker executor, фиксированную версию Docker этот вариант установки не требует ([релиз](https://docs.gitlab.com/releases/19/gitlab-19-3-released/), [Runner](https://docs.gitlab.com/runner/install/linux-repository/)).
+ Официальная базовая линия координатора: **8 vCPU, 16 ГБ RAM, ≥ 40 ГБ локального диска**; для репозиториев нужен дополнительный SSD по их объёму. Для отдельного Runner официального универсального CPU/RAM/HDD нет: ресурсы зависят от job; практический старт малого стенда — **2 vCPU, 4 ГБ RAM, 20 ГБ SSD**, затем ограничивать и измерять jobs ([requirements](https://docs.gitlab.com/install/requirements/), [Runner autoscaling](https://docs.gitlab.com/runner/runner_autoscale/)).
+ Свободны **443** (HTTPS UI/API и Runner), **22** (Git SSH), **80** (redirect/получение сертификата); внутренние **8075**, **5432**, **6379** наружу не открывать ([порты](https://docs.gitlab.com/administration/package_information/defaults/)).

Установка — выбран Linux package, не Helm ([официальный quickstart для Ubuntu](https://docs.gitlab.com/install/package/ubuntu/)):
+ Установить и пинить `gitlab-ee=19.3.0-ee.0`, задать свой `EXTERNAL_URL` и пароль `root`; затем установить Runner **19.3.0** и зарегистрировать токеном `glrt-`. Учебный all-in-one не является HA и не переносится в бой ([пин версии](https://docs.gitlab.com/update/package/), [регистрация Runner](https://docs.gitlab.com/runner/register/)).

Подключение ([CI/CD](https://docs.gitlab.com/ci/)):
+ Разработчики работают по HTTPS **443** или SSH **22**; Runner опрашивает API GitLab по HTTPS **443**. Секреты хранить в CI/CD variables/Vault, не в `.gitlab-ci.yml` ([job token](https://docs.gitlab.com/ci/jobs/ci_job_token/)).
