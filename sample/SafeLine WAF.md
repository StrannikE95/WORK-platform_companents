
Подготовка:
+ Одна выделенная Linux-машина x86_64 (виртуальная машина или железо) рядом с учебным Kubernetes
+ 2 CPU / 4 ГБ / 5 ГБ свободных на каталог /data/safeline на локальном диске. 
 На этой машине будет Docker Compose. Tengine (форк Nginx, контейнер safeline-tengine) работает в host-сети: слушает порты самой машины 
Поэтому на этой VM нельзя держать Ingress или другой процесс, которому тоже нужны 80 и 443. 
Не ставить IMAGE_TAG=latest.  
Свободны на хосте порты 80, 443, консоль 9443, служебные 65508 (health check, не меняется) и 65443 (своя страница ошибки, не меняется).
Установка:
https://docs.waf.chaitin.com/en/GetStarted/Deploy
Подключение:
https://docs.waf.chaitin.com/en/GetStarted/AddApplication



