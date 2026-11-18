# AI туслахын хариулт (засваргүй, үгчлэн)

AI-д өгсөн асуулт: шилжүүлгийн тодорхойлолтыг (зааврын `POST /transfer` хэсгийг бүтнээр) өгөөд
"Доорх шилжүүлгийн функцын тодорхойлолтоос category-partition аргаар сонголт (category), төлөөлөх утга,
хязгаарлалт ([ERROR], [SINGLE], [IF/property]) гаргаад Microsoft PICT модель бич." гэж хүссэн.
Миний A хэсэг болон моделийг AI-д үзүүлээгүй. Доорх моделийг `ai-transfer.model` нэрээр хадгалж,
CI-д ажиллуулсан (`results/ai-transfer.txt`).

---

## 1. Category-partition задаргаа

**Тэмдэглэгээ:** `[ERROR]` нь зөвхөн нэг алдааг шалгах утга (бусад параметр хэвийн байна), `[SINGLE]` нь хязгаарын утгыг ганц удаа шалгана, `[property X]` нь шинж чанар зарлана, `[IF X]` нь X шинж чанартай үед л хэрэглэгдэнэ.

| Категори (PICT параметр) | Сонголт (PICT утга) | Төлөөлөх утга (жишээ) | Хязгаарлалт | Хүлээгдэх үр дүн |
|---|---|---|---|---|
| **from данс** (`From`) | `Active` | `"A1"`: байгаа, царцаагдаагүй данс | `[property fromOK]` | цааш шалгана |
| | `~Frozen` | `"A2"`: байгаа боловч frozen | `[ERROR]` | 200 `ERROR_FROZEN` |
| | `~Missing` | `"X999"`: байхгүй ID | `[ERROR]` | 200 `ERROR_NO_FROM` |
| **to данс** (`To`) | `OtherExisting` | `"B1"`: байгаа, from-оос өөр | `[property toOK]` | цааш шалгана |
| | `~SameAsFrom` | `to = "A1"` | `[ERROR]` | 200 `ERROR_SAME_ACCOUNT` |
| | `~Missing` | `"Y999"`: байхгүй ID | `[ERROR]` | 200 `ERROR_NO_TO` |
| **amount тэмдэг** (`AmountSign`) | `Positive` | `100` | `[property positive]` | цааш шалгана |
| | `~Zero` | `0` (хязгаар) | `[ERROR]` | 200 `ERROR_BAD_AMOUNT` |
| | `~Negative` | `-50` | `[ERROR]` | 200 `ERROR_BAD_AMOUNT` |
| **currency** (`Currency`) | `Same` | данс MNT, currency `"MNT"` | `[property sameCur]` | хөрвүүлэлт хийхгүй |
| | `Different` | данс MNT, currency `"USD"` | `[property convert]` | ханшаар хөрвүүлнэ |
| **ханш** (`Rate`) | `None` | хөрвүүлэлтгүй | `[IF sameCur]` | — |
| | `GreaterThanOne` | USD→MNT (1 USD ≈ 3400 MNT): хөрвүүлсний дараа дүн өсдөг | `[IF convert]` | — |
| | `LessThanOne` | MNT→USD: хөрвүүлсний дараа дүн буурдаг | `[IF convert]` | — |
| **Хөрвүүлсэн дүн ба үлдэгдэл** (`Balance`) | `Below` | хөрвүүлсэн дүн < үлдэгдэл | `[IF positive]` | OK |
| | `Equal` | хөрвүүлсэн дүн = үлдэгдэл | `[SINGLE]` | 201 OK ("хэтрэхгүй" гэдэг нь тэнцүүг зөвшөөрнө) |
| | `~Above` | хөрвүүлсэн дүн > үлдэгдэл | `[ERROR]` | 200 `ERROR_INSUFFICIENT` |
| **Хөрвүүлсэн дүн ба өдрийн лимит** (`Limit`) | `Below` | өнөөдрийн зарцуулалт + дүн < лимит | `[IF positive]` | OK |
| | `Equal` | өнөөдрийн зарцуулалт + дүн = лимит | `[SINGLE]` | 201 OK |
| | `~Above` | өнөөдрийн зарцуулалт + дүн > лимит | `[ERROR]` | 200 `ERROR_LIMIT` |

**Бүх утга хэвийн үед:** 201 `{"result":"OK"}`.

**PICT рүү хэрхэн буулгасан бэ:**
- **`[ERROR]` → `~` угтвар.** PICT-ийн negative value нь нэг мөрөнд хамгийн ихдээ нэг байна. Бусад параметр нь заавал хэвийн утгатай гарна.
- **`[IF]` → `IF ... THEN ...` хязгаарлалт.**
- **`[SINGLE]` → хязгаарлалт.** PICT-д шууд `[SINGLE]` гэсэн тэмдэглэгээ байхгүй. Тиймээс `Equal` утгыг бусад параметрийн суурь утгатай (`Same`, `Below`) "түгжиж", хослолын тоог багасгасан.

**Тодорхойлолтод тодорхойгүй байгаа зүйлс (тайландаа дурдаарай):**
1. Хэд хэдэн алдаа зэрэг тохиолдвол аль нь түрүүлж буцахыг тодорхойлолт заагаагүй. Жагсаалтын дарааллаар (NO_FROM → … → LIMIT) гэж таамаглав. `~` ашигласан тул нэг мөрөнд хоёр алдаа гарахгүй.
2. Тодорхойлолтод байхгүй валютын код (`"XYZ"`) ирвэл ямар алдаа өгөх нь тодорхойгүй.
3. Өдрийн лимит нь өнөөдөр хийсэн өмнөх шилжүүлгүүдийг нийлүүлж тооцох уу гэдэг тодорхойгүй.
4. Хамгийн бага эерэг дүн (0.01) зөвшөөрөгдөх эсэх, бутархайн нарийвчлал тодорхойгүй.

## 2. PICT модель (`transfer.model`)

```text
# Category-partition model for POST /transfer
# ~ = [ERROR] value (PICT puts at most one negative value per test case)

From:       Active, ~Frozen, ~Missing
To:         OtherExisting, ~SameAsFrom, ~Missing
AmountSign: Positive, ~Zero, ~Negative
Currency:   Same, Different
Rate:       None, GreaterThanOne, LessThanOne
Balance:    Below, Equal, ~Above
Limit:      Below, Equal, ~Above

# [IF convert] / [IF sameCur]: exchange rate only matters when currency differs
IF [Currency] = "Same" THEN [Rate] = "None" ELSE [Rate] <> "None";

# [IF positive]: balance and limit checks are meaningful only for amount > 0
IF [AmountSign] <> "Positive" THEN [Balance] = "Below" AND [Limit] = "Below";

# [SINGLE]: boundary "Equal" values are tested only with base values of the others
IF [Balance] = "Equal" THEN [Limit] = "Below" AND [Currency] = "Same";
IF [Limit] = "Equal" THEN [Balance] = "Below" AND [Currency] = "Same";
```

**Ажиллуулах:** `pict transfer.model` (анхдагчаар pairwise), Excel-д оруулах бол `pict transfer.model > transfer_cases.txt`.

Гаралтын мөр бүрт `~` тэмдэгтэй утга байвал хүснэгтийн "Хүлээгдэх үр дүн" баганаас харгалзах `ERROR_...` хариуг авна. `~` тэмдэггүй мөр бол 201 `{"result":"OK"}` гарах ёстой.

Модель зөвхөн ASCII тэмдэгттэй. Учир нь PICT-ийн зарим хувилбар кирилл үсэг болон UTF-8 BOM-тэй файлыг буруу уншдаг.
