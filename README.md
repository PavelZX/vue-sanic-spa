Создаем приложение: Docker, VueJs и Python-Sanic
В рамках статьи мы создадим одностраничное (SPA) приложение с использованием VueJs. Оно будет общаться с WebSocket-сервером, предварительно авторизировавшись через backend api. WebSocket-сервер и api реализуем при помощи фреймворка Sanic.
В этом материале хочу поделиться наработками и изложить свое субъективное видение связки технологий, которая, как мне кажется, наиболее эффективна для fullstack разработчика с акцентом на Python. Постараюсь быть максимально кратким.
Кратко обо мне
Я вошел в веб 15 лет назад, во времена тотального доминирования PHP в качестве серверной платформы. Впоследствии я перешел на Python. Разработка web-приложений из монолитного кода начала «растекаться», сначало на серверную и интерфейсную части, потом стало модным растягивать по микросервисам еще и backend. Сейчас зачастую мне приходиться брать на себя роль бекенд- и фронтенд-разработчика. При этом сильно прокачиваются навыки DevOps, чтобы заставить кучу всего разрозненного работать вместе.
Стек, который я использую в работе сейчас:
•	Python framework: Flask; Sanic;
•	DB: PostgreSQL, Redis, RebbitMQ, MongoDB, MySQL;
•	JavaScript: VueJs, Jquery;
•	DevOps: Docker, Ansible.
Постановка задачи
Пишем приложение для http://localhost:
1.	По ссылке /db открывается web-клиент adminer для работы с PostgreSQL.
2.	По ссылке /app открывается web-клиент на VueJs, который содержит форму авторизации и после ее успешного выполнения позволяет получать/отправлять сообщения через webSocket.
3.	По ссылке /api реализовать API (python-Sanic):

o	авторизовывать пользователя из PostgreSQL;
o	данные сессии записывать в Redis.
4.	По ссылке /ws реализовать WebSocket server, принимающий запросы от авторизованного через API клиента и отвечающий ему.
Аргументация выбранного стека
Почему Redis? — Тут без комментариев.
Почему PostgreSQL? — Благодаря огромному опыту работы с MySQL, эмпирическим путем пришел к наиболее оптимальному решению.
Почему VueJs? — Не хотелось бы священных войн, изложу кратко свое видение. Для меня, как для fullstack разработчика с упором на backend, VueJs оказался весьма прост в освоении и понятен в своей концепции. Более популярные фреймворки React и особенно Angular имеют, на мой взгляд, несопоставимо более высокий порог вхождения. Они требуют гораздо больше времени на изучение и удержание достаточного уровня квалификации.
Почему backend на Sanic? — В среде Python в последнее время произошла довольно серьезная революция, связанная с развитием асинхронного программирования. У меня был опыт написания кода на Twisted и Tornado, но современная реализация async/await выше всяких похвал. Как грибы после дождя появились различные асинхронные веб-фреймворки. Но мой выбор пал на Sanic, потому как он практически полностью копирует интерфейсность Flask (в котором у меня наибольший опыт работы последних лет), при этом добавляет приложению огромную производительность и функциональность (тот же WebSockets).
Нюанс: Sanic еще весьма молод, под него мало пакетов. Производятся попытки миграции c Flask такого полезного расширения, как Restfull. На момент написания статьи там еще далеко даже до бета-релиза. Но, по моему мнению, сам фреймворк стабилен и великолепен — можно пользоваться.
Реализация и вводные данные
Мой рабочий компьютер:
	cat /etc/os-release | grep PRETTY_NAME  && docker -v && docker-compose -v && git --version &&  make -v

	PRETTY_NAME="Linux Mint 18"
	Docker version 18.06.1-ce, build e68fc7a
	docker-compose version 1.23.2, build 1110ad01
	git version 2.7.4
	GNU Make 4.1
view rawarticle-1-before hosted with ❤ by GitHub
Этап 1. Инициализация проекта
	#!/bin/bash
	# Выполняем из консоли
	git init vue-sanic-spa;
	cd vue-sanic-spa;
	# В каталог .data контейнеры Redis и Postgresql будут создавать свои базы данных(поговорим об этом чуть ниже)
	mkdir .data;
	# Создаем директорию в которой будет "жить" код серверной части приложения
	mkdir api;
	# Создаем директорию для frontend приложения на vue
	mkdir app;
	# Создаем файл переменных окружения
	touch .env
	# Создаем и наполняем .gitignore
	touch .gitignore;
	# формируем правила попадания файлов в git-репозиторий
	echo '.*' >> .gitignore
	echo '!/.gitignore' >> .gitignore
	echo '!/.env' >> .gitignore
	# Создаем файл docker-compose.yml
	touch docker-compose.yml; 
view rawstage_1.sh hosted with ❤ by GitHub
Этап 2. Как это все будет работать
Согласно постановки задачи, у нас есть 2 приложения: Redis, PostgreSQL, которые должны быть изолированы от доступа извне, но должны быть доступны для использования из Python. C другой стороны, есть 4 точки входа, которые коммуницируют с внешним миром, то есть доступны через браузер. Это /app, /api, /ws, /db -за каждым из этих интерфейсов стоит свой работающий процесс.
Для /app — это процесс webpack watcher, задача которого — остлеживать изменение кода, который мы будем вносить, и налету преобразовывать его при помощи подключенных плагинов (broserify, cssminify и т. д.) в исполняемый браузером код.
Для /api и /ws — это будет один общий python worker, который также будет отслеживать изменения и перезапускать самого себя.
Все вышеописанные правила довольно просто реализовать при помощи Nginx, проксируя URL-запросы к им соответствующим демонам, которые работают на выделенных портах.
Этап 3. Контейнеризация
Поместим в файл .env, созданный ранее, несколько переменных окружения, которые понадобятся нам при построении контейнеров:
	ADMINER_DEFAULT_SERVER=pgsql
	POSTGRES_DB=dev
	POSTGRES_USER=dev
	POSTGRES_PASSWORD=test
	PYTHONASYNCIODEBUG=1
	PYTHONDONTWRITEBYTECODE=1
	API_MODE=dev
	API_ADMIN_EMAIL=admin@example.com
	API_ADMIN_PASSWORD=password
	API_WEBSOCKET_TIMEOUT=86400
view raw.env hosted with ❤ by GitHub
Обращаю внимание, что этот файл лишь в рамках этой статьи был добавлен в Git-репозиторий. В реальной жизни ему не место в репозитории. На сервере этот файл создается ручками, со своими значениями переменных. Далее:
	# Создаем дирректорию nginx и в ней файл конфигурации сервера
	mkdir nginx && touch nginx/server.conf 
	# внутрь которого вставляем нижеслудующий код:

	server {
	    listen 80;
	    server_name localhost;   
	    
	    location /db {            
	        rewrite /db$     /db    break;  
	        rewrite /db/(.*) /db/$1  break;  
	        proxy_redirect     off;
	        proxy_set_header   Host                 $host;
	        proxy_set_header   X-Real-IP            $remote_addr;
	        proxy_set_header   X-Forwarded-For      $proxy_add_x_forwarded_for;
	        proxy_set_header   X-Forwarded-Proto    $scheme;
	        proxy_set_header Host $http_host;
	        proxy_pass http://adminer:8080;
	    }    
	}
view rawarticle-1-stage-2 hosted with ❤ by GitHub
В конфигурации сервера мы декларируем запуск Nginx на 80-м порту. При этом все запросы, URL которых начинается с /db, перебрасываем на сервис adminer-а, который будет запущен на порту 8080. Я вынесу за рамки статьи информацию о том, что такое Docker, как связан с ним docker-compose и как это все работает. Ознакомиться с этими приложениями можно в официальной документации.
Перейдем непосредственно к реализации. На момент написания статьи версия 3.7 — последняя для спецификации содержимого docker-compose.yml. Ее и будем использовать:
	version: '3.7'
	networks:
	  web:
	    driver: bridge
	  internal:
	    driver: bridge

	services:  
	  redis:
	    container_name: test_redis
	    image: "redis:latest"
	    volumes:
	      - ${PWD}/.data/redis:/data        
	    networks:
	      - internal

	  db:
	    restart: always
	    container_name: test_db
	    image: postgres:alpine
	    
	    environment:
	      POSTGRES_DB: ${POSTGRES_DB}
	      POSTGRES_USER: ${POSTGRES_USER}
	      POSTGRES_PASSWORD: ${POSTGRES_PASSWORD}
	    networks:
	      - internal
	    volumes:
	      - ${PWD}/.data/postgresql:/var/lib/postgresql/data
	      - ${PWD}/db_entrypoint:/docker-entrypoint-initdb.d

	  adminer:
	    container_name: test_adminer
	    image: adminer    
	    restart: always
	    env_file:
	      - .env
	    networks:
	      - internal  

	  nginx:
	    container_name: test_nginx
	    image: "nginx:stable"      
	    ports:
	      - "80:80"
	    volumes:
	      - ./nginx:/etc/nginx/conf.d
	    networks:
	      - web
	      - internal
	
	    env_file:
	      - .env
	    links:       
	      - adminer     
view rawdocker-compose.yml hosted with ❤ by GitHub
Cети (networks)
Строки 2-6: мы определили 2 вида сетей, через которые будут коммуницировать наши контейнеры. Необходимость и назначение сетей я оставлю за рамками этой статьи, благо есть исчерпывающая документация.
Создавать разные сети для Dev-окружения совершенно нет необходимости. Но, как уже говорил выше, для одного разработчика хранить и актуализировать параллельно 2 сценария контейнеризации — излишняя трата времени. В рамках ограниченных ресурсов необходимо стараться делать окружение таким, каким оно будет выглядеть в продакшене с минимальными изменениями. Также можно оспорить необходимость делать веб-доступ к базе данных продакшена через adminer. Локально я предпочитаю коннектиться к базе через консоль, но иногда для оперативности — веб-доступ. Это экономит время.
Redis
Строки 9-15: стандартные инструкции создания Redis контейнера. Обратите внимание, что доступ к нему другие контейнеры могут получить из подсети «internal».
PostgreSQL
Строки 18-30: создаем контейнер на базе официального образа PostgreSQL. Обратите внимание на строки 23-25. По умолчанию перед запуском docker-compose автоматически читает переменные, заданные в .env файле, но не передает их дальше в контейнер для создания базы данных, пользователя и пароля. Для этой цели существует 2 способа. Первый — через переменную сервиса environment, где необходимо задать пару ключ-значение. Второй способ — через переменную env_file, когда мы пробрасываем все переменные файла .env непосредственно в контейнер (сделаем это чуть позже на примере серверного приложения).
В строке 29 мы «заставляем» базу данных хранить данные вне контейнера.
В строке 30 мы в момент создания контейнера запускаем SQL-инструкцию по созданию базы данных «test», которую будем использовать при написании тестов.
Adminer
Строки 33-39: поднимаем оффициальный Docker контейнер от Adminer. Обращаем внимание, что в конфигурации нужно явно указать сервис(ы), к которым он будет иметь доступ (в нашем случае это DB). Также указываем internal подсеть, по которой будет идти коммуникация.
Отступление. Наряду с Adminer на продакшене возможна контейнеризация любого другого множества вспомогательных приложений (скажем, redis-commander или phpMyAdmin). Рекомендую поднимать их в отдельной подсети, с доступом к ней только из-под VPN. Можно не заморачиваться с VPN, а ограничить доступ к определенным сервисам по IP, но такой подход менее гибок.
Строки 41-50: запускаем контейнер на базе официального образа Nginx, пробрасывая наш ранее созданный конфигурационный файл. Обратите внимание: необходимо указать связь с контейнером adminer через подсеть internal. Без этого указанный в server.conf сервис http://adminer:8080 доступен не будет.
В следующих частях статьи рассмотрим, как расширить файл server.conf, дополнить для работы с двумя другими сервисами app и api.
Этап 4. Построение и запуск контейнеров
На этапах 1-3 мы полностью подготовили необходимое для разработки окружения. Осталось запустить все и проверить, как это работает.
	# из консоли запускаем
	docker-compose up  -d
	

view rawarticle-1-stage-4 hosted with ❤ by GitHub
Теперь:
	#Команда
	docker ps
	
	#вернет нам приблизительно следующие данные:
	CONTAINER ID IMAGE COMMAND CREATED STATUS PORTS NAMES
	4e0299752062 redis:latest "docker-entrypoint.s…" 45 minutes ago Up 45 minutes 6379/tcp test_redis
	52e79174ff42 nginx:stable "nginx -g 'daemon of…" 12 hours ago Up 45 minutes 0.0.0.0:80->80/tcp test_nginx
	cbe035df98d7 adminer "entrypoint.sh docke…" 12 hours ago Up 45 minutes 8080/tcp test_adminer
	d0a380622c85 postgres:alpine "docker-entrypoint.s…" 12 hours ago Up 45 minutes 5432/tcp test_db
view rawarticle-1-stage-41 hosted with ❤ by GitHub
Открываем в браузере http://localhost/db и видим страницу входа в Adminer. Логинимся в базу данных при помощи значений переменных файла .env. Убеждаемся, что выполнился скрипт из директории db_entrypoint/, создавший базу test.
Итог
Весь исходный код я выложил на GitHub в ветку master. Следующая часть, чтобы можно было сравнить изменения, будет сделана в отдельной ветке. Чтобы покрутить самостоятельно, выполните из консоли:
	cd ~
	git clone git@github.com:v-kolesov/vue-sanic-spa.git
	cd vue-sanic-spa
	docker-compose up  -d
view rawarticle-1e1-epilogue hosted with ❤ by GitHub
После чего пробуйте зайти на localhost/db.
Итак, уточним задачу, которую предстоит реализовать в API:
1.	Пусть новое приложение по ссылке /api/v1.0/user/auth принимает POST запрос с параметрами username и password и, в случае совпадения данных доступа с учетной записью в таблице users, возвращает token идентификации.
2.	Данные об авторизованном пользователе (users.id) пишем в Redis, где ключом (key) будет token, созданный в пункте 1.
Этап 1. Небольшая автоматизация рутины
Во время разработки я довольно часто пользуюсь возможностями утилиты make, позволяющей намного сократить «вербальность» некоторых команд. Для этого в корне проекта создаю Makefile с набором удобных инструкций-сокращений. Приведу список наиболее полезных для меня:
	include .env
	up:
		docker-compose up -d
	upb:
		docker-compose up -d --force-recreate --build
	stop:
		docker-compose stop
	db:
		export PGPASSWORD=${POSTGRES_PASSWORD}; docker exec -it test_db psql -U $(POSTGRES_USER) ${POSTGRES_DB}
	r:
		docker exec -it test_redis  /usr/local/bin/redis-cli
	test:
		docker exec -it test_api pytest 
	b:
		docker exec -it $(c) /bin/bash
view rawarticle-2-stage-0 hosted with ❤ by GitHub
make up — просто запускает, а make upb — еще и полностью перестраивает перед запуском все существующие контейнеры.
make stop — останавливает все работающие контейнеры.
make db — подключаемся к БД PostgreSQL при помощи консольного клиента pgSQL. Обратите внимание, что мы в первой строчке Makefile «заинклюдили» все переменные окружения, которые у нас есть в .env файле. Поэтому здесь могут быть довольно экзотические инструкции, список которых ограничен лишь фантазией разработчика. К примеру, у меня ещё здесь находятся инструкции по деплою кода на продакшен при помощи ansible).
make r — быстрый способ посмотреть значения в базе данных Redis через консольную утилиту redis-cli.
make test — запускает unit/integrity tests, которые нам еще предстоит написать для нашего /api.
Последняя значимая make-инструкция — это (пример) make b c=test_api — подключиться к bash-консоли любого контейнера с явно указанным именем.
Этап 2. Создаем минимально рабочее API
Итак:
	# создаем файл с указанием необходимых нам в работе пакетов
	touch api/requirements.txt

	echo 'sanic' >> api/requirements.txt
	echo 'asyncio_redis' >> api/requirements.txt
	echo 'asyncpg' >> api/requirements.txt
	echo 'passlib' >> api/requirements.txt
	echo 'pytest' >> api/requirements.txt
	echo 'marshmallow' >> api/requirements.txt
	echo 'aiohttp' >> api/requirements.txt
view rawarticle-2-E1 hosted with ❤ by GitHub
Перед тем как мы начнем реализовать логику API, нам необходимо сделать минимально работающее Sanic-приложение, контейнеризировать его и настроить проксирование всех запросов, начинающихся на /api в контейнере с nginx на наш новый контейнер. Для этого «утащим» пример hello world отсюда и слегка модифицируем его. В каталоге api нашего проекта, создаем файл run.py со следующим содержимым:
	import os
	from sanic import Sanic
	from sanic.response import json

	app = Sanic()
	
	@app.route("/")
	async def test(request):
	    return json({"hello": "world"})
	

	if __name__ == "__main__":                
	    debug_mode =  os.getenv('API_MODE', '') == 'dev'   
	
	    app.run(
	        host='0.0.0.0',
	        port=8000,
	        debug=debug_mode, 
	        access_log=debug_mode, 
	        auto_reload=debug_mode
	    )
view rawarticle-2-stage-20 hosted with ❤ by GitHub
Теперь нам нужно запустить run.py внутри контейнера. Для этого создаем api/Dockerfile со следующим содержимым:
	FROM python:3.6.7-slim-stretch
	WORKDIR /app
	RUN apt-get update
	RUN apt-get -y install gcc
	COPY requirements.txt /tmp
	RUN pip install -r /tmp/requirements.txt
	VOLUME [ "/app" ]
	EXPOSE 8000
	CMD ["python", "run.py"]
view rawarticle-2-stage-21 hosted with ❤ by GitHub
В этом файле мы указываем компоновщику, что для создания нашего контейнера нужно взять последний Docker образ Python версии 3.6.7. Далее создаем рабочий каталог /app для нашего микросервиса.
Отступление. Я взял за правило помещать весь рабочий код приложения в корневую папку /app контейнера. Аналогично будет и в примере с клиентским app, которое будет реализовано в следующей части.
Далее идет установка дополнительного пакета gcc, без которого не получится инсталлировать некоторые pip-пакеты из api/requirements.txt, которые мы указали чуть выше, при его создании. Далее ничего особенного, перейдем к настройке файла docker-compose.yml.
Этап 3. Добавляем api в docker-compose.yml
	  api: 
	    container_name: test_api
	    build: 
	      context: ./api
	    tty: true
	    restart: always
	    volumes: 
	      - "./api:/app"    
	    links:
	      - "db" 
	      - "redis"     
	    networks:      
	      - internal
	    env_file:
	      - .env
	    ports:
	      - "8000:8000"
view rawarticle-2-3-0 hosted with ❤ by GitHub
Здесь в 4-й строке мы «заставляем» docker-compose изменить контекст и пройти по указанному адресу для чтения инструкций, созданному нами во 2-м этапе «Dockerfile», который создает контейнер с работающим приложением. Особо важной для нас, как для разработчиков, является директива tty, которая позволяет подключаться (docker attach) к консоли вывода работающего процесса в контейнере. Эта жизненно необходимая для dev разработки инструкция позволит нам «дебажить» и «тюнинговать» наше приложение.
Кроме того, мы указали что наш сервис «связан» (links) c контейнерами db и redis через подсеть (network) internal.
В строчке 7 мы «пробрасываем» папку api «внутрь» контейнера (механизм volumes).
В строчке 14, как я уже рассказывал в первой части, мы пробрасываем и делаем доступными для использования приложением переменных окружения из файла .env, созданного в прошлой части. Одна из этих переменных (API_MODE), уже используется в run.py.
Этап 4. Изменяем конфигурацию nginx
Всё, что нам осталось сделать, — добавить правила проксирования для сервиса nginx. Для этого отредактируем файл nginx/server.conf, добавив:
	    location /api {            
	        rewrite /api$     /    break;  
	        rewrite /api/(.*) /$1  break;  
	        proxy_redirect     off;
	        proxy_set_header   Host                 $host;
	        proxy_set_header   X-Real-IP            $remote_addr;
	        proxy_set_header   X-Forwarded-For      $proxy_add_x_forwarded_for;
	        proxy_set_header   X-Forwarded-Proto    $scheme;
	        proxy_set_header Host $http_host;
	        proxy_pass http://api:8000;
	    }
view rawarticle-2-4 hosted with ❤ by GitHub
Теперь пересоздадим контейнеры с учетом новых изменений.
Этап 5. Остановка и перестройка контейнеров
С учетом вышеописанного в статье, у нас всё готово для запуска:
	# из корня нашего проекта запускаем
	make stop
	make upb
	

	# или, если нравится традиционный способ
	docker-compose stop
	docker-compose up -d --force-recreate --build
	

view rawarticle-2-5 hosted with ❤ by GitHub
В браузере вбиваем http://localhost/api, результатом должно быть {"hello":"world«}
Этап 6. Как этим пользоваться
Благодаря переменной окружения API_MODE=dev из файла .env, мы запустили приложение в режиме debug. Фреймворк sanic, как и множество других Python Web-фреймворков, поддерживает hot-reload. То есть сервер приложения автоматически будет перезапускаться, как только мы изменим один из файлов, связанных с run.py. Проверить это довольно легко: исправьте «world» на «world!» в коде run.py и тут же обновите страницу браузера. Вы увидите, что информация обновилась.
Но перезагрузки кода не достаточно. Нам необходимо «дебажить» код и где-то видеть логи приложения. Для этого существуют две команды:
	docker attach test_api
	docker logs test_api
view rawarticle-2-stage-60 hosted with ❤ by GitHub
Первая из них подключается к консольному выводу работающего процесса (чтобы отключиться Ctrl+C), куда мы, при помощи стандартного пакета logging (или просто print), можем выводить любую debug-информацию. Вторая команда необходима для тех случаев, когда мы ловим fatal, «не совместимый с жизнью самого процесса». Вывод этой команды показывает нам backtrace, предшествовавший ошибке.
На заметку. Я выше обращал внимание на инструкцию tty=true. Как раз она позволяет делать «безболезненный» detach по Ctrl+C, не уничтожая сам процесс.
Этап 7. Авторское отступление
Лично я, когда программирую backend, то консоль с выводом (docker attach test_api) у меня открыта в терминальном окне редактора кода. Еще, для поддержки технологии intellisense, которая есть во многих популярных редакторах, я внутри папки api создаю виртуальное окружение .venv, в которое устанавливаю те же пакеты из requirements.txt, что установлены в самом контейнере:
	 python3 -m venv api/.venv && source api/.venv/bin/activate && pip install -r api/requirements.txt 
view rawarticle-2-70 hosted with ❤ by GitHub
В корневой директории в файле .gitignore указана инструкция пропуска всех папок и файлов, начинающихся с «.», поэтому можно не переживать, что «мусор» попадет в репозиторий.
Этап 8. Реализация логики
В рамках статьи невозможно делать полный копипаст кода, который я написал (много файлов и большой объем), поэтому, как говорил в прошлой статье, будет ссылка на pull request, где можно посмотреть изменения, которые были детально описаны выше. Прокомментирую наиболее важные моменты.
Был добавлен файл api/application.py, содержащий функцию-фабрику, которую будем использовать в 2-х местах: в run.py для сервера приложения и в conftest.py — для создания экземпляра (fixture) приложения, для написания интеграционного теста. Хотел бы обратить особое внимание на функцию db_migrate. Фактически она делает то же самое, что и расширение alembic, которое помогает делать миграцию схемы БД между различными ветками кода. Я реализовал схему миграции следующим образом:
	import os
	from passlib.handlers.pbkdf2 import pbkdf2_sha256
	

	LATEST_VERSION = 2
	

	SCHEMA = {
	    2: {
	        'up': [
	            ["""
	                CREATE TABLE users(
	                 id BIGSERIAL PRIMARY KEY,
	                 email VARCHAR (64) UNIQUE NOT NULL,
	                 password TEXT NOT NULL,
	                 created_on TIMESTAMP default (now() at time zone 'utc'),
	                 confirmed BOOLEAN default false
	                );"""
	             ],
	            ["INSERT INTO users (email, password) values ($1, $2)",
	                os.getenv('API_ADMIN_EMAIL'),
	                pbkdf2_sha256.hash(os.getenv('API_ADMIN_PASSWORD'))
	             ],
	            ["CREATE INDEX idx_users_email ON users(email);"],
	        ],
	        'down': [
	            ["DROP INDEX idx_users_email"],
	            ["DROP TABLE users;"],
	        ]
	    },
	    1: {
	        'up': [
	            ["CREATE TABLE versions( id integer NOT NULL);"],
	            ["INSERT INTO versions VALUES ($1);", 1]
	        ],
	        'down': [
	            ["DROP TABLE versions;"],
	        ]
	    }
	}
view rawmigration.py hosted with ❤ by GitHub
Немного пояснений. В момент перезапуска сервера система «смотрит» в БД в поисках значения таблицы version. Если такой таблицы нет, текущая версия определяется как «0» и приложение автоматически запускает все скрипты из ветки «up» до тех пор, пока значение ключа из SCHEMA не станет равным LATEST_VERSION. Последняя успешная версия фиксируется в version. Обратный процесс аналогичен. Изменяя значение LATEST_VERSION, функция миграции «спускается» вниз на любую версию. При этом она запускает все скрипты из «down», вплоть до полной очистки базы данных, при условии, если LATEST_VERSION указываем равным «0».
Еще хотелось бы обратить внимание на то, что доступ к версиям api реализован при помощи механизма blueprint, который выбран из-за возможной версионности api в будущем.
На заметку. Когда пишем интеграционные тесты, то тестируем URL локального сервера, то есть тестируем /v1.0/user/auth. Но когда в следующей статье перейдем к реализации frontend, мы будем обращаться к «api» через прокси-сервер nginx в формате /api/v1.0/user/auth.
Этап 9. Запуск
	Если уже делали чекаут проекта, то
	# Подтягиваем изменения в проект
	git pull
	git checkout master
	# останавливаем и пересоздаем контейнеры
	make stop
	make upb
	# запускаем тесты для приложения api
	make test
	

	#Для тех, кто только присоединился
	cd ~
	git clone git@github.com:v-kolesov/vue-sanic-spa.git
	cd vue-sanic-spa
	make upb
	make test
view rawarticle-2-9 hosted with ❤ by GitHub
Уточним, что нам осталось сделать согласно постановки задачи:
1.	Реализовать WebSocket сервер на http://localhost/ws; (используем Python-Sanic).
2.	Написать простейший чат http://localhost/ (используя VueJs) который будет авторизироваться через http://localhost/api, реализованный в Части 2. Получим некий token, при помощи которого можно будет подключиться к чату на базе WebSocket-сервера из п. 1.
Этап 1. WebSocket server на Sanic
Небольшое отступление. Асинхронный фреймворк Sanic позволяет реализовать WS-сервер на базе созданного еще во 2-й части API. Я решил создать отдельный процесс, чтобы, с одной стороны, не смешивать код из разных частей статьи, с другой, чтобы наглядно продемонстрировать простоту микросервисной архитектуры.
Итак:
# Из корня проекта выполняем:
  mkdir ws
Добавляем файл ws/Dockerfile
FROM python:3.6.7-slim-stretch
WORKDIR /app
RUN apt-get update
RUN apt-get -y install gcc
COPY requirements.txt /tmp
RUN pip install -r /tmp/requirements.txt
VOLUME [ "/app" ]
EXPOSE 8000
CMD ["python", "run.py"]
Здесь мы «просим» docker создать нам контейнер, взяв за основу образ с python:3.6.7. Нужно доустановить в него некоторые системные библиотеки и необходимые для работы приложения python-пакеты из ws/requirements.txt:
# содержимое ws/requirements.txt
sanic
asyncio_redis
Кроме того, мы указали, что запуск приложения будет осуществляться на 8000 порту внутри самого контейнера, а команда реализация сервера находиться внутри run.py:
import os
from time import time
from sanic import Sanic
import ujson
import asyncio_redis
from websockets.exceptions import ConnectionClosed

app = Sanic('websocket')

conn = {}
CONN_CACHE_TIME = 10 # sec

@app.listener('before_server_start')
async def start(app, loop):
    app.redis = await asyncio_redis.Pool.create(host='redis', poolsize=10)


@app.listener('after_server_stop')
async def stop(app, loop):    
    app.redis.close()


async def check_token(request):
    token = request.args['token']   
    

async def checkTokenAlive(ws, token):
    if time() - conn.get(ws, 0) > CONN_CACHE_TIME:
        token_exists = await app.redis.exists(token)
        if token_exists:
            conn[ws]=time()            
        else:
            return False
    return True
    

@app.websocket('/')
async def feed(request, ws):                      
    token = request.args['token'].pop()
    if token:
        isAlive = await checkTokenAlive(ws, token)
        while isAlive:
            try:
                data = await ws.recv()            
                if data:                
                    if data=="/out":
                        await ws.close()
                    await ws.send(f'I\'ve received: {data}')                            
            except  ConnectionClosed:
                pass
            isAlive = await checkTokenAlive(ws, token)
    await ws.close()


if __name__ == "__main__":                
    debug_mode =  os.getenv('API_MODE', '') == 'dev'   

    app.run(
        host='0.0.0.0',
        port=8000,
        debug=debug_mode, 
        access_log=debug_mode
    )
В сервере предусмотрена периодическая (10 секунд) проверка token на «наличие» в Redis. Напомню, данный токен — это то, что получит наше SPA в случае успешной авторизации на localhost/api/v1.0/user/auth (реализация в Часть 2). Дополнительно реализована инструкция «/out» для самого чата, которая при отправке с клиента, закрывает текущее WebSocket-соединение.
Дополняем docker-compose.yml новым сервисом:
services:    
  ws: 
    container_name: test_ws
    build: 
      context: ./ws
    tty: true
    restart: always
    volumes: 
      - "./ws:/app"    
    links:      
      - "redis"     
    networks:      
      - internal
    env_file:
      - .env
Корректируем конфигурацию nginx сервера таким образом, чтобы все запросы, которые приходят на localhost/ws, проксировались к контейнеру «ws»:
services:    
       location /ws {            
        rewrite /ws$     /    break;  
        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection "upgrade";        
        proxy_redirect     off;
        proxy_set_header   Host                 $host;
        proxy_set_header   X-Real-IP            $remote_addr;
        proxy_set_header   X-Forwarded-For      $proxy_add_x_forwarded_for;
        proxy_set_header   X-Forwarded-Proto    $scheme;
        proxy_set_header Host $http_host;
        proxy_pass http://ws:8000;
    }
Так как мы пишем SPA-приложение, работающее в браузере, нам необходимо получить самый что ни есть «классический» JavaScript, который гарантированно будет выполняться на подавляющем большинстве браузеров, установленных у пользователей. В силу очевидных причин, развитие JavaScript как языка программирования (ES5, ES6 и т. д.) сильно ушло вперед по сравнению с тем, что могут предложить существующие браузеры. По этой причине для того, чтобы воспользоваться всей мощью языка на клиенте, нам необходимо «преобразовать» наш «современный» синтаксис в («старый») понятный для браузера код.
Для этой цели как нельзя лучше подходит пакетный статический анализатор кода Webpack, к которому в качестве плагинов можно подключить конвертеры: Browserify, TypeScript, Sass, Less, css-minify, js-minify и многие другие, облегчающие web-разработку клиента. В общем случае — Webpack представляет собой демон (работающий процесс), который в зависимости от конфигурации отслеживает изменение кода и «налету» преобразовывает «удобный и современный» js-код в аналогичный по функционалу, но подходящий для работы в большинстве браузеров. В общем случае текст кода, который мы пишем, последовательно преобразовывается подключаемыми к демону плагинами и сохраняется в виде одного/двух файлов в директории /dist.
Настройка Webpack (установка и конфигурирование плагинов) — весьма муторное занятие, но существуют «утилиты-помощники», позволяющие выполнить эту затратную по времени операцию. Так как мы пишем фронтенд на VueJs, то в качестве такого «помощника» воспользуемся утилитой vue-cli, которая, кроме разворачивания Webpack, создаст базовое тестовое приложение на Vue, которое мы изменим под свои цели.
Этап 2. Создаем «SPA demo» при помощи vue-cli
Итак, нам нужно:
•	запустить контейнер с Node (версии LTS 10.5);
•	сгенерировать при помощи vue-cli demo-приложение;
•	запустить его в контейнере;
•	настроить маршрутизацию запросов сервера nginx.
Выполним из корня нашего проекта команды:
    # Переменной GID присваиваем идентификатор группы хост-машины
    echo "GID=$(id -g)" >> .env
    # Переменной GID присваиваем идентификатор текущего пользователя
    echo "UID=$(id -u)" >> .env
Cоздаем файл app/Dockerfile:
# За основу выбераем последний стабильный (LTS) образ версии Node
FROM node:10.15.0-alpine
WORKDIR /app
VOLUME ["/app"]
# инсталируем vue-cli согласно https://cli.vuejs.org/guide/installation.html
RUN npm install -g @vue/cli
В docker-compose.yml добавляем:
services:
  app:
    build: ./app
    tty: true
    user: "${UID}:${GID}"
    container_name: test_app
    volumes:
      - "./app:/app"
    networks:
    - internal    
    env_file:
      - .env
Хочу обратить внимание на 5-ю строчку в сервисе app. Предварительно мы сохранили в файл .env ID пользователя и группы хостмашины в силу особенности устройства ядра Linux. Даём указание docker запустить контейнер от лица этого юзера и группы. Это сделано для того, чтобы файлы, которые мы будем сейчас создавать внутри контейнера, можно было редактировать/создавать извне, то есть из хостмашины, не меняя владельца (chown) всех файлов.
Далее выполняем:
# из корня нашего проекта, перестраиваем и перезапускаем контейнеры
make upb
# или, если нравится традиционный способ, то:
docker-compose up -d --force-recreate --build
Мы запустили контейнер с Node версии 10.15.0, c предварительно установленным vue-cli.
Теперь:
  # подключаемся к sh-консоли работающего контейнера c Node  (в docker-compose его имя test_app)
docker exec -it test_app /bin/sh
# в консоли контейнера переходим в корневой каталог
cd /
# создаем demo приложение при помощи уже установленной во время создания контейнера vue-cli
vue create app
# Мастер, сообщаем что директория не пуста.. Выбираем "Merge"
# Затем в  меню "Manually select features" выбираем пакеты которые бы мы хотели установить для нашего приложения
# Выбираем нужные пакеты, я оставил:
# ◉ Babel
# ◉ Router
# ◉ CSS Pre-processors
# ◉ Linter / Formatter
# Далеее, на все вопросы установщика, можем соглашаться по умолчанию.
# В конце установки пакетов, мастер выдает сообщение
# $ cd app
# $ npm run serve
Выполнив последние 2 команды, мы увидим, что webpack по умолчанию запустился на 8080 порту.
Отключаемся от контейнера (Ctrl+С) и видим, что на хост-машине (ls -la app/ ) в папке фронтенда контейнер сгенерил «кучу» файлов, которые благодаря механизму Volumes, теперь являются общими для хост-машины и для контейнера. Самое интересное здесь то, что благодаря GID и UID сгенеренные из контейнера файлы принадлежат текущему юзеру host-машины, хотя внутри контейнера юзер имеет другое имя. Более исчерпывающую информацию о пользователях/группах в контейнерах можно почерпнуть здесь.
Так как у нас уже есть готовый рабочий код, все, что нам осталось сделать, — это подправить наш app/Dockerfile таким образом, чтобы контейнер при запуске выполнял команду запуска демона, то есть npm run serve.
# окончательный вид app/Dockerfile
FROM node:10.15.0-alpine
RUN apk add --no-cache bash
RUN npm install -g @vue/cli
WORKDIR /app
VOLUME ["/app"]
RUN npm install 
EXPOSE 8080
CMD ["npm", "run", "serve"]
Этап 2.1. Настраиваем nginx для vue-приложения
В конец конфигурационного файла nginx/server.conf вставим:
location / {                    
        rewrite /(.*) /$1  break;          
        proxy_redirect     off;
        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection "upgrade";
        proxy_set_header   Host                 $host;
        proxy_set_header   X-Real-IP            $remote_addr;
        proxy_set_header   X-Forwarded-For      $proxy_add_x_forwarded_for;
        proxy_set_header   X-Forwarded-Proto    $scheme;
        proxy_set_header Host $http_host;
        proxy_pass http://app:8080;
    }
Теперь перезапускаем все созданные контейнеры:
make upb
и заходим на http://localhost. Должны увидеть demo-страницу vue.
Важно! Demo-приложение, которое генерит vue-cli, по умолчанию для mode dev подключает интегрированный плагин vue-hot-reload-api, который через WebSocket-соединение «узнает» от демона webpack об изменениях на сервере и автоматически перезагружает страницу приложения в браузере, подтягивая новые данные. По этой причине, в конфигурации nginx, заложена поддержка проксирования WebSocket-заголовков.
  # Если нужно видеть вывод консоли работающего WebPack, 
  # цепляемся к контейнеру командой
docker attach test_app
Этап 2.2. Пишем клиентское приложение
Немного поговорим о том, как будет работать наш SPA-клиент:
1.	При заходе на localhost проверяем, есть ли в localStorage значение для ключа «token». Если есть, пробуем установить WebSocket-соединение с ws://localhost/ws?token=xxx с сервером. Если сервер закрыл соединение (token отсутствует в redis), сбрасываем приложение на страницу /login.
2.	На странице /login находится форма авторизации, которая отправляет данные на api (http://localhost/api/v1.0/user/auth), и в случае успеха сохраняет token в localStorage с последующим редиректом на главную.
3.	В случае неудачной авторизации, отрисовываем повторно форму авторизации с отображением ошибки валидации.
К сожалению, объем кода SPA-приложения довольно большой, что неминуемо приведет к тому, что объем статьи будет огромный. Все изменения кода в этой статье, я выполнил в отдельной ветке на GitHub. Их можно посмотреть в виде pull request, которые я выполнил по сравнению с состоянием кода по окончании 2-й части.
Итоговый результат
Если вы дочитали до этого места, то, пожалуй, именно сейчас как раз тот момент, когда вы можете полностью увидеть реализованный вариант рабочего приложения, состоящего из 7 контейнеров.
  cd ~
git clone git@github.com:v-kolesov/vue-sanic-spa.git
cd vue-sanic-spa
docker-compose up  -d
После запуска всех контейнеров в браузере вбейте http://localhost. Вы должны увидеть, нечто похожее. В моем случае, в правом нижнем углу — 2 терминала, которые подключены непосредственно к контейнерам test_app, test_ws (нужно в целях отладки и дебагинга).

