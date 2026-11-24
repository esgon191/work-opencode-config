#work-opencode-config
Конфинг для корпоративного агента для работы с набором скиллов
 
## exports
### Путь к конфигу:
```sh
export OPENCODE_CONFIG_DIR=~/Desktop/work-opencode-config/
```

### Креды для моделей:: 
```sh
export DEEPSEEK_API_KEY=
export GLM_API_KEY= 
```

### Креды для тулов:: 
```sh
export JIRA_PERSONAL_TOKEN= 
export CONFLUENCE_PERSONAL_TOKEN= 
export RAG_OPERSMKT_API_KEY="<ключ LiteLLM>"
```

`RAG_OPERSMKT_API_KEY` нужен для подключения к RAG через LiteLLM. Отдельный токен RAG
задавать не требуется: он скрыт внутри LiteLLM. В OpenCode инструменты
`rag-code-oper-smkt_*` разрешены только сабагенту `searcher`; для остальных агентов
они отключены глобально в `opencode.jsonc`. Не добавляй реальные токены в репозиторий.
