---
name: java-dto-to-openapi
# Описание одновременно объясняет назначение и условия применения скилла.
description: Преобразует исходный код Java DTO в схему объекта OpenAPI 3.0 YAML. Используй, когда пользователь присылает Java DTO и просит получить components.schemas с именем без суффикса Dto.
---

# Преобразование Java DTO в OpenAPI Schema

## Задача

По присланному исходному коду Java DTO сгенерируй YAML-описание схемы объекта для `components.schemas` OpenAPI 3.0.3. Работай только с кодом и контекстом, которые предоставил пользователь. Не требуй доступа к репозиторию или дополнительных файлов для простых случаев.

## Формат результата

- Выводи **только YAML**, без пояснений и Markdown-обрамления, если пользователь не попросил иное.
- Выводи одну или несколько схем **на уровне элементов `components.schemas`**, без оберток `openapi:`, `info:`, `paths:`, `components:` и `schemas:`.
- Имя схемы получай из имени Java-класса, удаляя конечный суффикс `Dto` или `DTO` (регистр остальной части сохраняй): `MyDto` → `My`, `CreateOrderRequestDto` → `CreateOrderRequest`. Если суффикса нет, оставляй имя класса.
- Сохраняй порядок полей из Java-кода.
- Используй отступы в два пробела, без табуляции.
- Для каждого обычного DTO создавай `type: object` и `properties:`. Не добавляй пустые `properties`, если полей нет.

## Алгоритм

1. Определи имя основного DTO и публичные сериализуемые свойства: поля, объявленные в DTO, и, если явно показаны, свойства из родительских классов. Не включай `static`, `transient`, `@JsonIgnore` и вычисляемые методы, не являющиеся свойствами JSON.
2. Учитывай аннотации Jackson для именования и сериализации: например, `@JsonProperty("order_id")` изменяет имя свойства на `order_id`. При противоречии Java-имени и JSON-имени выбирай JSON-имя.
3. Преобразуй типы по таблице ниже. Учитывай тип элемента коллекций.
4. Применяй предоставленные аннотации валидации и документации, не придумывая их значения.
5. Вложенные DTO представляй через `$ref`, если они названы, но их реализация не предоставлена. Если реализации вложенных DTO также даны, добавляй их схемы рядом с основной, с именами по тем же правилам.
6. Проверяй, что итоговый YAML синтаксически корректен и что использованные `$ref` либо определены во входных данных, либо явно относятся к внешним, не предоставленным схемам.

## Соответствие типов

| Тип Java | OpenAPI YAML |
| --- | --- |
| `String`, `char`, `Character`, `UUID` | `type: string` (`format: uuid` для UUID) |
| `byte`, `short`, `int`, `Integer`, `Short`, `Byte` | `type: integer`, `format: int32` |
| `long`, `Long`, `BigInteger` | `type: integer`, `format: int64` для `long`/`Long`; для `BigInteger` используй `type: integer` без формата |
| `float`, `Float` | `type: number`, `format: float` |
| `double`, `Double` | `type: number`, `format: double` |
| `BigDecimal` | `type: number` без `format` |
| `boolean`, `Boolean` | `type: boolean` |
| `LocalDate` | `type: string`, `format: date` |
| `Instant`, `OffsetDateTime`, `ZonedDateTime` | `type: string`, `format: date-time` |
| `LocalDateTime` | `type: string` (без `date-time`, если соглашение о часовом поясе не задано) |
| `List<T>`, `Set<T>`, `T[]` | `type: array`, `items:` по типу `T` |
| `Map<String, T>` | `type: object`, `additionalProperties:` по типу `T` |
| Java `enum` | `type: string`, `enum:` со значениями, только если они предоставлены |
| Другой именованный DTO | `$ref: '#/components/schemas/ИмяБезDto'` |

Если тип неизвестен или параметризирован непонятно, не выдумывай поля и значения: обозначь свойство как `{}` и, если пользователь допускает пояснения, сообщи о неопределенности. Тип `Optional<T>` раскрывай как `T`; сам по себе `Optional` не делает поле обязательным.

## Обязательность и ограничения

- **Не считай поле обязательным только потому, что оно имеет примитивный тип Java** или объявлено `final`.
- Добавляй поле в `required`, если оно помечено `@NotNull`, `@NotBlank`, `@NotEmpty` или `@JsonProperty(required = true)`. `required` — это присутствие свойства в JSON, а не запрет `null`.
- `@Size(min = ..., max = ...)` преобразуй в `minLength`/`maxLength` для строк, `minItems`/`maxItems` для коллекций. `@Min`, `@Max`, `@DecimalMin`, `@DecimalMax` переводятся в `minimum` и `maximum` там, где применимо.
- `@Pattern(regexp = "...")` → `pattern` с корректным экранированием YAML.
- `@Schema(description = "...")` → `description`, `@Schema(example = "...")` → `example` с учетом типа. Не придумывай описания или примеры.
- Если из аннотаций явно следует, что `null` разрешен, применяй `nullable: true` в OpenAPI 3.0; отсутствие `@NotNull` само по себе **не доказывает** разрешение `null`.
- Если обязательных свойств нет, не добавляй `required: []`.
- Не создавай `nullable: false` по умолчанию: в OpenAPI 3.0 отсутствие `nullable: true` уже запрещает `null` для указанного типа.
- Если `enum` упоминается, но константы не показаны, используй `type: string` без выдуманного списка `enum`.
- Не определяй бизнес-ограничения, значения по умолчанию и обязательность по имени поля или догадке.

## Пример 1. Простой DTO

**Вход: `MyDto.java`**

```java
import jakarta.validation.constraints.NotBlank;
import jakarta.validation.constraints.Min;

public class MyDto {
    @NotBlank
    private String name;
    private Long id;
    @Min(0)
    private Integer count;
    private boolean active;
}
```

**Результат:**

```yaml
My:
  type: object
  required:
    - name
  properties:
    name:
      type: string
    id:
      type: integer
      format: int64
    count:
      type: integer
      format: int32
      minimum: 0
    active:
      type: boolean
```

## Пример 2. Коллекция, вложенный DTO и переименование свойства

**Вход: `OrderDto.java` и `OrderItemDto.java`**

```java
import com.fasterxml.jackson.annotation.JsonProperty;
import jakarta.validation.constraints.NotNull;
import jakarta.validation.constraints.Size;
import java.math.BigDecimal;
import java.time.LocalDate;
import java.util.List;
import java.util.UUID;

public class OrderDto {
    @NotNull
    private UUID id;

    @JsonProperty("customer_name")
    private String customerName;

    @Size(min = 1)
    private List<OrderItemDto> items;

    private LocalDate deliveryDate;
}

public class OrderItemDto {
    @NotNull
    private String sku;
    private BigDecimal price;
}
```

**Результат:**

```yaml
Order:
  type: object
  required:
    - id
  properties:
    id:
      type: string
      format: uuid
    customer_name:
      type: string
    items:
      type: array
      minItems: 1
      items:
        $ref: '#/components/schemas/OrderItem'
    deliveryDate:
      type: string
      format: date

OrderItem:
  type: object
  required:
    - sku
  properties:
    sku:
      type: string
    price:
      type: number
```

## Пример 3. Enum

**Вход: `PaymentDto.java`**

```java
public class PaymentDto {
    private PaymentStatus status;
}

enum PaymentStatus {
    NEW, PAID, FAILED
}
```

**Результат:**

```yaml
Payment:
  type: object
  properties:
    status:
      type: string
      enum:
        - NEW
        - PAID
        - FAILED
```

## Приоритет правил

Если пользователь задает дополнительные правила преобразования, соблюдай их в первую очередь. Если входной код неполон, сформируй максимально точную схему по доступной информации, не выдавая предположения за факты. Не добавляй endpoint'ы и REST-операции: задача ограничена схемами объектов.
