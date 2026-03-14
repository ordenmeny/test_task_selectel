1) Баг 1
При запуске проекта через Docker произошла ошибка:
 File "/usr/local/lib/python3.11/site-packages/pydantic/main.py", line 250, in __init__
app-1  |     validated_self = self.__pydantic_validator__.validate_python(data, self_instance=self)
app-1  |                      ^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^
app-1  | pydantic_core._pydantic_core.ValidationError: 1 validation error for Settings
app-1  | database_url
app-1  |   Extra inputs are not permitted [type=extra_forbidden, input_value='postgresql+asyncpg://pos...stgres@db:5432/postgres', input_type=str]

Контейнер с приложением не стартует, т.к. проблема с настройками приложения, где pydantic-settings заметил ошибку.

Я открыл файл с настройками core.config и увидел, что для параметра validation_alias значение DATABSE_URL. Я открыл .env и увидел DATABASE_URL=..., т.е. имя поля не совпадает, т.е. значение из .env не подтягивается.

Решение:
Заменил DATABSE_URL на DATABASE_URL, теперь поля из .env и настроек совпадают.
database_url: str = Field(
    "postgresql+asyncpg://postgres:postgres@db:5432/postgres_typo",
    validation_alias="DATABASE_URL",
)

Также неверное значение по умолчанию:
"postgresql+asyncpg://postgres:postgres@db:5432/postgres_typo",
Должно быть не postgres_typo, а postgres для текущих переменных POSTGRES_DB: postgres (из docker-compose.yml), но это не влияет на запуск приложения.


2) Баг 2
Ошибка возникает во время запуска приложения

app-1  | 2026-03-14 05:40:47,010 | ERROR | app.main | Ошибка фонового парсинга: 'NoneType' object has no attribute 'name'
app-1  | Traceback (most recent call last):
app-1  |   File "/app/app/main.py", line 24, in _run_parse_job
app-1  |     await parse_and_store(session)
app-1  |   File "/app/app/services/parser.py", line 43, in parse_and_store
app-1  |     "city_name": item.city.name.strip(),
app-1  |                  ^^^^^^^^^^^^^^
app-1  | AttributeError: 'NoneType' object has no attribute 'name'


Причина: поле city может быть None. А если оно None, но у него не может быть никаких атрибутов.
class ExternalVacancyItem(BaseModel):
	...
    city: Optional[ExternalCity]
    ...

Решение: добавить проверку через тернарный оператор.
parsed_payloads.append(
    {
 		...
        "city_name": item.city.name.strip() if item.city else None,
        ...
    }
)

3) Баг 3
Парсинг выполняется не каждые 5 минут, а каждые 5 секунд, это видно из логов Docker:
app-1  | 2026-03-14 06:20:11,229 | INFO | app.services.parser | Старт парсинга вакансий
app-1  | 2026-03-14 06:20:11,783 | INFO | httpx | HTTP Request: GET https://api.selectel.ru/proxy/public/employee/api/public/vacancies?per_page=1000&page=1 "HTTP/1.1 200 OK"
app-1  | 2026-03-14 06:20:11,808 | INFO | app.services.parser | Парсинг завершен, новых вакансий: 0
app-1  | 2026-03-14 06:20:11,809 | INFO | apscheduler.executors.default | Job "_run_parse_job (trigger: interval[0:00:05], next run at: 2026-03-14 06:20:16 UTC)" executed successfully

Причина: apscheduler требует передавать время в секундах, а передается в минутах.
Решение:
scheduler.add_job(
	...
    seconds=settings.parse_schedule_minutes * 60,
    ...
)	

4) Баг 4
При текущей конфигурации SettingsConfigDict используется extra='forbid', т.к. оно идёт по умолчанию. Если в .env появятся дополнительные переменные, то приложение упадёт с ValidationError. Заметил это, когда подключал pgadmin4 для работы с БД через docker-compose и указал в .env следующее:
PGADMIN_DEFAULT_EMAIL=myemail@yandex.ru
PGADMIN_DEFAULT_PASSWORD=admin123

pydantic_core._pydantic_core.ValidationError: 2 validation errors for Settings
app-1              | pgadmin_default_email
app-1              |   Extra inputs are not permitted [type=extra_forbidden, input_value='myemail@yandex.ru', input_type=str]
app-1              |     For further information visit https://errors.pydantic.dev/2.12/v/extra_forbidden
app-1              | pgadmin_default_password
app-1              |   Extra inputs are not permitted [type=extra_forbidden, input_value='admin123', input_type=str]

Решение: добавить параметр extra="ignore":
model_config = SettingsConfigDict(
    env_file=".env",
    env_file_encoding="utf-8",
    case_sensitive=False,
    extra="ignore",
)

5) Баг 5
В файле crud/vacancy.py в функции upsert_external_vacancies неверный тип для existing_ids.
Сначала идет работа с existing_ids как со списком, потом переводится в множество, но если попадаем в else, то изменяется на словарь, а должно быть множество.

Решение:
изменить existing_ids = {} на existing_ids = set()

6) Баг 6
В файле crud/vacancy.py в функции upsert_external_vacancies при формировании списка external_ids. Если external_id будет равно 0, то текущее условие не возьмет его в выборку.
Решением будет заменить external_ids = [payload["external_id"] for payload in payloads if payload["external_id"]] на это:

external_ids = [
    payload.get("external_id")
    for payload in payloads
    if payload.get("external_id") is not None
]

Также если external_id будет равно 0, то не пройдет и проверка:
if ext_id and ext_id in existing_ids:
	...

Решение:
заменить на 
if ext_id is not None and ext_id in existing_ids:
	...

7) Баг 7 в endpoint POST / в файле api/v1/vacancies.
Текущий ендпоинт:
@router.post("/", response_model=VacancyRead, status_code=status.HTTP_201_CREATED)

В декораторе указано, что endpoint возвращает объект VacancyRead и статус 201 (Created).
Однако если вакансия с таким external_id уже существует, возвращается другой ответ — 200 OK и JSON {"detail": ...}.

Это является багом, потому что нарушается контракт API: на клиенте ожидается VacancyRead, но в некоторых случаях приходит другой формат ответа. Из-за этого могут возникнуть проблемы на клиенте, ошибки валидации, некорректная OpenAPI-документация.

Решение: если вакансия уже существует, лучше выбрасывать HTTPException со статусом 409 Conflict, либо изменить логику endpoint так, чтобы он всегда возвращал определенный формат ответа.

raise HTTPException(
    status_code=status.HTTP_409_CONFLICT,
    detail="Vacancy with external_id already exists",
)

8) Баг 8
В services/parser.py в функции parse_and_store httpx.AsyncClient не закрывается.

Cоздаётся HTTP-клиент:
client = httpx.AsyncClient(timeout=timeout)
Но после выполнения запросов клиент должен закрываться, т.к. если клиент не
закрывается, то соединения могут оставаться открытыми дольше и со временем это может привести к накоплению открытых соединений.

Решение:
после завершения работы клиента его необходимо явно закрыть.
async def parse_and_store(session: AsyncSession) -> int:
	...
	client = None

	try:
	    client = httpx.AsyncClient(timeout=timeout)
	    ...
	except (httpx.RequestError, httpx.HTTPStatusError) as exc:
	    logger.exception("Ошибка парсинга вакансий: %s", exc)
	    return 0
	finally:
	    if client is not None:
	        await client.aclose()

	...


9) Баг 9 при endpoint PUT /vacancy_id (файл v1/vacancies.py)
При изменении external_id на существующий получим ошибку 500.
Например:
есть вакансия id=25, external_id=1517
есть вакансия id=11, external_id=1630

и обновить вакансию id=11, передав external_id=1517, то код дойдёт до commit(), а там база выбросит ошибку 500 из-за уникальности external_id.


Решением (app/api/v1/vacancies.py):
@router.put("/{vacancy_id}", response_model=VacancyRead)
async def update_vacancy_endpoint(
    vacancy_id: int,
    payload: VacancyUpdate,
    session: AsyncSession = Depends(get_session),
) -> VacancyRead:
    vacancy = await get_vacancy(session, vacancy_id)
    if not vacancy:
        raise HTTPException(status_code=status.HTTP_404_NOT_FOUND, detail="Not found")

    if payload.external_id is not None:
        existing = await get_vacancy_by_external_id(session, payload.external_id)
        if existing and existing.id != vacancy.id:
            raise HTTPException(
                status_code=status.HTTP_409_CONFLICT,
                detail="Vacancy with external_id already exists",
            )

    return await update_vacancy(session, vacancy, payload)

Т.е. добавлена проверка уникальности external_id перед обновлением.
