## 2. Схема таблиц базы данных (PostgreSQL)

### 2.1 Склады (`warehouses`)
_В alpha не используется, по-умолчанию создается один склад `Мой склад`_
```sql
CREATE TABLE public.warehouses (
    id        serial CONSTRAINT warehouses_pk PRIMARY KEY, 
    name      varchar(100)          NOT NULL -- название склада
)
```

### 2.1. Зоны склада (`zones`)
_По-умолчанию создается 3 зоны_
```sql
CREATE TABLE public.zones (
    id              serial CONSTRAINT zones_pk PRIMARY KEY,
    name            varchar(100)          NOT NULL,
    type            varchar(20)           NOT NULL,  -- 'RECEIVING' | 'STORAGE' | 'DISPATCH' (далее: 'QUARANTINE', 'DEFECT')
    is_system       boolean DEFAULT false NOT NULL,  -- true для базовых системных зон (защита от удаления)
    warehouse_id    integer NOT NULL                 -- ссылка на склад, которому принадлежит зона
);
```

### 2.2. Ячейки / Адреса хранения (`cells`)
```sql
CREATE TABLE public.cells (
                              id              bigserial CONSTRAINT cells_pk PRIMARY KEY, -- идентификатор ячейки
                              name            varchar(50)   DEFAULT ''::character varying NOT NULL, -- произвольное наименование
                              whs_id          integer       DEFAULT 0                     NOT NULL, -- идентификатор склада, при наличии нескольких складов
                              zone_id         integer       DEFAULT 0                     NOT NULL, -- идентификатор зоны
                              passage_id      integer       DEFAULT 0                     NOT NULL, -- проезд
                              rack_id         integer       DEFAULT 0                     NOT NULL, -- стеллаж
                              section_id      integer       DEFAULT 0                     NOT NULL,
                              floor           integer       DEFAULT 0                     NOT NULL, -- этаж
                              number          integer       DEFAULT 0                     NOT NULL, -- номер ячейки
    -- ВГХ (задел на v1.2, в v1.0 используются дефолты)
                              sz_length       integer       DEFAULT 0                     NOT NULL, -- длина ячейки
                              sz_width        integer       DEFAULT 0                     NOT NULL, -- ширина ячейки
                              sz_height       integer       DEFAULT 0                     NOT NULL, -- высота ячейки
                              sz_volume       numeric(8, 3) DEFAULT 0                     NOT NULL, -- объем ячейки
                              sz_uf_volume    numeric(8, 3) DEFAULT 0                     NOT NULL, -- полезный объем ячейки
                              sz_weight       numeric(8, 3) DEFAULT 0                     NOT NULL, -- допустимый вес для ячейки
                              is_size_free    boolean       DEFAULT false                 NOT NULL, -- безразмерная ячейка
                              is_weight_free  boolean       DEFAULT false                 NOT NULL, -- с любым весом
    -- Управляющие и сервисные флаги
                              not_allowed_in  boolean       DEFAULT false                 NOT NULL, -- запрет размещения
                              not_allowed_out boolean       DEFAULT false                 NOT NULL, -- запрет отбора
                              is_service      boolean       DEFAULT false                 NOT NULL  -- служебная ячейка
);

CREATE INDEX idx_cells_zone_service ON public.cells (zone_id, is_service);
CREATE INDEX idx_cells_hierarchy ON public.cells (whs_id, zone_id, passage_id, rack_id, section_id, floor, number);
```

### 2.3. Контейнеры / Тара (`containers`)
```sql
CREATE TABLE public.containers (
    id        bigserial CONSTRAINT containers_pk PRIMARY KEY,
    name      text,
    type      integer DEFAULT 0 NOT NULL,
    parent_id integer DEFAULT 0 NOT NULL, -- 0 = внешний контейнер (стоит непосредственно в cell_id)
    root_id   integer,                    -- ID корневого контейнера всей вложенной структуры
    depth     integer DEFAULT 0 NOT NULL  -- Уровень вложенности (0 = внешний)
);

CREATE INDEX idx_containers_root ON public.containers (root_id);
CREATE INDEX idx_containers_parent ON public.containers (parent_id);
```

### 2.4. Номенклатура (`items`)
```sql
CREATE TABLE public.items (
    id              bigserial CONSTRAINT items_pk PRIMARY KEY,
    name            text                                   NOT NULL, -- Поиск через pg_trgm
    manufacturer_id integer,                                         -- Идентификатор справочника производителей
    sku             text,
    barcode         text UNIQUE,                                     -- Заводской EAN-13 / Code128
    tags            text[]      DEFAULT '{}'               NOT NULL, -- Теги вместо справочника категорий
    attributes      jsonb       DEFAULT '{}'::jsonb        NOT NULL, -- Кастомные поля (цвет, бренд и т.д.)
    created_at      timestamptz DEFAULT now()              NOT NULL,
    updated_at      timestamptz DEFAULT now()              NOT NULL
);

CREATE INDEX idx_items_name_trgm ON public.items USING gin (name gin_trgm_ops);
CREATE INDEX idx_items_tags ON public.items USING gin (tags);
CREATE INDEX idx_items_attributes ON public.items USING gin (attributes);
```

### 2.5. Производители (`manufacturers`)
```sql
CREATE TABLE public.manufacturers (
    id              serial CONSTRAINT manufacturers_pk PRIMARY KEY,
    name            text          NOT NULL
);
CREATE INDEX idx_manufacturers_name_trgm ON public.manufacturers USING gin (name gin_trgm_ops);
```

### 2.6. Единый регистр движений (`storage`)
```sql
CREATE TABLE public.storage (
    row_id       varchar(36)              DEFAULT ''::character varying NOT NULL,
    row_time     timestamp with time zone DEFAULT now()                 NOT NULL,
    zone_id      integer, -- идентификатор зоны склада
    cell_id      integer, -- идентификатор ячейки
    container_id integer, -- идентификатор контейнера
    item_id      integer, -- идентификатор номенклатуры (теоретически допускаю, что здесь может быть null, например для учета контейнеров. в alpha не делаем)
    quantity     integer
);

CREATE INDEX idx_storage_balance ON public.storage (zone_id, cell_id, container_id, item_id) INCLUDE (quantity);
CREATE INDEX idx_storage_item ON public.storage (item_id) INCLUDE (zone_id, cell_id, container_id, quantity);
CREATE INDEX idx_storage_container ON public.storage (container_id) INCLUDE (zone_id, cell_id, item_id, quantity);
CREATE INDEX idx_storage_row_id ON public.storage (row_id);
```

### 2.7. Подсистема печати (CUPS-агент), Пользователи и Идемпотентность
```sql
CREATE TABLE public.printers (
    id           serial PRIMARY KEY,
    agent_id     text                  NOT NULL,
    cups_name    text                  NOT NULL,
    display_name text                  NOT NULL,
    is_online    boolean DEFAULT true  NOT NULL,
    last_seen_at timestamptz DEFAULT now() NOT NULL,
    UNIQUE (agent_id, cups_name)
);

CREATE TABLE public.print_jobs (
    id                  bigserial PRIMARY KEY,
    printer_id          integer,
    printer_type        integer, -- 'A4', 'TERMO'
    payload_type        varchar(10) NOT NULL, -- 'PDF' | 'PNG' | 'RAW'
    payload_data        bytea       NOT NULL, -- 
    entity_type_id      integer, -- агент сам может формировать этикетки, по типу данных (вместо payload_data)
    entity_id           integer, -- и идентификатору данных
    copies              integer DEFAULT 1 NOT NULL,
    status              varchar(20) DEFAULT 'PENDING' NOT NULL, -- 'PENDING' | 'PRINTED' | 'FAILED'
    error_text          text,
    created_at          timestamptz DEFAULT now() NOT NULL
);

CREATE TABLE public.users (
    id                  serial PRIMARY KEY,
    username            varchar(50) UNIQUE NOT NULL,
    password_hash       text               NOT NULL,
    role                varchar(20)        NOT NULL, -- 'admin' | 'worker' | 'viewer'
    def_printer_id      integer,  -- принтер по умолчанию для печати на обычной бумаге
    def_tprinter_id     integer   -- термопринетр
);

CREATE TABLE public.operations_inbox (
    key           varchar(64) PRIMARY KEY,
    response_code integer     NOT NULL,
    response_body jsonb       NOT NULL,
    created_at    timestamptz DEFAULT now() NOT NULL
);
```
