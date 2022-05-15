# Настройка PostgreSQL

https://pgtune.leopard.in.ua

1) Настройка доступа
	1) Идем в файл pg_hba
2) Главный файл настройки postgresql.conf. Расположен по умолчанию в `Ubuntu - /etc/postgresql/14/main/postgresql.conf` 
Посмотреть через psql `show config_file;`
3) pg_settings и pg_file_settings
	1) источник информации о параметрах и их значениях
	2) от куда эти параметры берутся
	3) как эти параметры могут изменятся  применятся
	4) это более удобный аналог show