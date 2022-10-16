
Playbook
---
Playbook устанавливает и конфигурирует `clickhouse` и `vector`
#### Clickhouse
- установка
- создание БД
#### Vector
- установка
#### Действия
Разворачивает Clickhouse и Vector.
 
Переменные
---
Можно отредактировать `clickhouse/vars_yml`. Можно изменить параметры:
`clickhouse_version` - версия `clickhouse`

Можно отредактировать `vector/vars_yml`. Можно изменить параметры:
`ansible_architecture` - версия `vector`

Тэги:
---
- `clickhouse` - установка и конфигурирование `clickhouse` 
- `vector` - установка `vector`