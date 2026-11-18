# Лаборатори №6 — Тест тохиолдол боловсруулах: Category-Partition ба PICT

**Оюутан:** Эрдэнэбаяр
**Оюутны код:** B232270136
**Хичээл:** F.CSA313 — Программ хангамжийн чанарын баталгаа ба тест (2026)

---

## 1. Ажлын орчин ба PICT-ийг эх кодоос бүтээсэн нь

Миний машин: Windows 11 Pro (26200). Энд **Smart App Control** enforcement горимд
(`HKLM\SYSTEM\CurrentControlSet\Control\CI\Policy\VerifiedAndReputablePolicyState = 1`) ажиллаж байна.
Гарын үсэггүй exe-г "An Application Control policy has blocked this file" гэж хаадаг. Энэ хоригт
pip-ээр суулгасан `cmake.exe`, `zig.exe` хаагдсан бөгөөд эх кодоос бүтээсэн `pict.exe` ч мөн ажиллахгүй.
WSL ч суугаагүй.

Тиймээс PICT-ийг лабын заасан **Ubuntu** орчинд, зааварт бичсэн командуудаар эх кодоос нь бүтээсэн.
Бүтээлтийг GitHub Actions-ийн `ubuntu-24.04` runner дээр
[`.github/workflows/pict.yml`](.github/workflows/pict.yml) хийдэг:

```bash
sudo apt-get install -y git cmake g++ make
mkdir -p ~/tools && cd ~/tools && git clone https://github.com/microsoft/pict && cd pict   # репоноос ГАДНА
cmake -S . -B build -DCMAKE_BUILD_TYPE=Release
cmake --build build -j4
sudo cp build/cli/pict /usr/local/bin/
(git -C ~/tools/pict log -1 --format='pict commit %h %ad' --date=short; pict 2>&1 | head -3) | tee results/pict-build.txt
```

- PICT-ийн эх код, `build/` нь runner-ийн `~/tools/pict`-д үлдэх тул энэ репод ордоггүй.
  `.gitignore`-д `pict/` ба `build/`-ийг мөн нэмсэн.
- PICT-ийг `ab76c25` commit дээр тогтоосон тул дахин ажиллуулах бүрт ижил гаралт гарна.
- Workflow нь репо дахь бүх `*.model`-ийг ажиллуулж `results/*.txt`-ийг дахин үүсгэдэг.
  Дараа нь commit хийгдсэн `results/`-тэй `git diff --exit-code`-ээр тулгадаг. Ингэснээр README ба
  `results/`-ийн тоо PICT-ийн бодит гаралттай таарч байгааг push бүрд шалгана.
- `results/*.txt` файлуудыг workflow-ийн artifact-аас татаж аваад өөрөө commit хийсэн.

---

## 2. Хэсэг A — Category-Partition гараар: `POST /transfer`

**Тестийн бэхэлгээ (setup), бүх хүснэгтэд нийтлэг:**
- from данс `ACC-100`, валют **MNT**;
- өдрийн лимит **L = 1 000 000 MNT**;
- ханш **1 USD = 3 400 MNT**.

Тэмдэглэгээ: **x** = хөрвүүлсний дараах дүн (MNT), **B** = from-ийн үлдэгдэл, **R** = өнөөдөр үлдсэн лимит (L − өнөөдөр шилжүүлсэн).

### A1 — Сонголтууд ба төлөөлөх утгууд

| Параметр | Сонголт (category) | Утга | Төлөөлөх утга / setup | Хязгаарлалт (A2) |
|---|---|---|---|---|
| from | from дансны төлөв | F1 Valid | `ACC-100` байгаа, идэвхтэй | [property hasAccount] |
| | | F2 Frozen | `ACC-200` байгаа, царцаагдсан | [ERROR] [property hasAccount] |
| | | F3 Missing | `ACC-999` (системд байхгүй) | [ERROR] |
| to | to дансны төлөв | T1 Valid | `ACC-300` байгаа, from-оос өөр | |
| | | T2 Missing | `ACC-998` (байхгүй) | [ERROR] |
| | | T3 SameAsFrom | to = from | [ERROR] [if hasAccount] |
| | | T4 Frozen | `ACC-400` байгаа, царцаагдсан | [SINGLE] |
| amount | x-ийг B ба R-тэй харьцуулсан байдал | M1 Normal | 0 < x < min(B, R), ж: 10 000 | [if hasRemaining] |
| | | M2 Zero | 0 | [ERROR] |
| | | M3 Negative | −5 000 | [ERROR] |
| | | M4 EqualBalance | x = B (B < R), ж: B = 200 000, x = 200 000 | [SINGLE] [if hasAccount] [if hasRemaining] |
| | | M5 OverBalance | x = B + 1 (≤ R) | [ERROR] [if hasAccount] |
| | | M6 EqualLimit | x = R (R < B), ж: R = 400 000, x = 400 000 | [SINGLE] [if hasAccount] [if hasRemaining] |
| | | M7 OverLimit | x = R + 1 (≤ B) | [ERROR] [if hasAccount] |
| currency | from дансны валюттай харьцуулбал | C1 Same | `MNT` | |
| | | C2 Different | `USD` → ×3 400 хөрвүүлнэ | [property converted] [if hasAccount] |
| *(далд орчин)* | өнөөдөр from-оос аль хэдийн шилжүүлсэн | L1 None | 0 → R = 1 000 000 | [property hasRemaining] [if hasAccount] |
| | | L2 Partial | 600 000 → R = 400 000 | [property hasRemaining] [if hasAccount] |
| | | L3 Full | 1 000 000 → R = 0 | [ERROR] [if hasAccount] |

**Далд орчны сонголт:** Хүсэлтийн body-д харагдахгүй ч үр дүнг тодорхойлдог 3 зүйл бий:
1. **өдрийн лимитээс аль хэдийн хэр ашигласан** (L1–L3). Ижил amount=400 000 нь L1 үед OK, L2 үед
   яг лимит, L3 үед ERROR_LIMIT болно.
2. **from-ийн үлдэгдэл B.** M4/M5-ийн setup-аар тогтоосон.
3. **ханш.** C2-ийн setup-аар тогтмол 3 400 болгосон. Ханш өөрчлөгдөхөд хил хаашаа шилжихийг
   тодорхойлолт хэлдэггүй.

### A2 — Хязгаарлалтууд ба үндэслэл

- **[ERROR]** — F2, F3, T2, T3, M2, M3, M5, M7, L3. Тус бүр тодорхойлолтод заасан нэг ERROR_ код
  буцаах ёстой. Нэг алдаа бусдыгаа нуудаг (error masking) тул бусад сонголттой хослуулах нь тест
  нэмэхгүй. Тиймээс тус бүрийг **ганц удаа** шалгана.
- **[SINGLE]**:
  - **M4, M6** нь "хэтрэхгүй" (≤) гэсэн хилийн утга. Тэнцүү үед зөвшөөрөгдөх эсэхийг нэг тест л
    тогтооно, бусадтай хослуулах шаардлагагүй.
  - **T4** (to царцаагдсан) нь тодорхойлолтод **огт дурдагдаагүй**. Шууд утгаар уншвал 201 OK байх ёстой
    тул нэг удаа шалгаж, шаардлагын эзэнтэй тодруулах асуулт болгоно.
- **[property hasAccount] / [if hasAccount]** — Байхгүй from данс (F3)-д үлдэгдэл, лимит, валют
  байхгүй тул B, R, "Different" гэсэн ойлголт **утгагүй**. Мөн "to = from" нь байхгүй данстай адил болно.
  Тиймээс M4–M7, C2, T3, L1–L3 зөвхөн from данс байгаа үед утгатай.
- **[property hasRemaining] / [if hasRemaining]** — L3 (R = 0) үед "0 < x < R" (M1),
  "x = B < R" (M4), "x = R" (M6) **гаргах боломжгүй**: R = 0 бол x = 0 нь Zero болно.
- **[property converted]** — C2 нь хөрвүүлэлтийн замыг идэвхжүүлнэ. Хөрвүүлсний дараах дүнгээр
  B/R-ийг давах сонирхолтой тохиолдлыг A3-ын S19 мөрөнд тусад нь гаргасан.

**Тоо (Лекц 6-ын find жишээний маягаар):**

```
Бүрэн олонлог : 3 (from) × 4 (to) × 7 (amount) × 2 (currency) × 3 (limit used) = 504
Хязгаарлалтын дараа:
  [ERROR]  : F2, F3, T2, T3, M2, M3, M5, M7, L3            = 9 frame (тус бүр 1)
  [SINGLE] : T4, M4, M6                                     = 3 frame (тус бүр 1)
  Хэвийн   : F1 × T1 × M1 × {C1, C2} × {L1, L2} = 1·1·1·2·2 = 4 frame
  Нийт     : 9 + 3 + 4                                       = 16
=> 504 → 16
```

### A3 — Спецификацийн хүснэгт

**Олон алдааны давуу эрэмбэ (миний тогтоосон):**
`NO_FROM → FROZEN → NO_TO → SAME_ACCOUNT → BAD_AMOUNT → INSUFFICIENT → LIMIT`.

**Үндэслэл:** Шалгалт бүр өмнөхөө давсан байхыг шаарддаг. Хамаарлын дарааллаар:
- B ба R-тэй харьцуулахын тулд from данс байгаа, идэвхтэй, дүн нь хүчинтэй байх ёстой;
- "to = from" шалгалт хоёр данс хоёулаа олдсоныг шаарддаг;
- илгээгчийн асуудлыг хүлээн авагчийнхаас өмнө шалгана;
- бодит үлдэгдэл (INSUFFICIENT) нь бодлогын лимитээс (LIMIT) өмнө.

Энэ нь тодорхойлолт дахь алдааны жагсаалтын дараалалтай ч таарна.

| # | from | to | amount (x) | currency | limit used | Хүлээгдэх үр дүн | Тайлбар |
|---|---|---|---|---|---|---|---|
| S1 | F1 Valid | T1 Valid | M1 10 000 | C1 MNT | L1 None | **201 OK** | happy path |
| S2 | F1 Valid | T1 Valid | M1 10 000 | C1 MNT | L2 Partial | **201 OK** | хэвийн хослол |
| S3 | F1 Valid | T1 Valid | M1 10 USD (=34 000) | C2 USD | L1 None | **201 OK** | хөрвүүлэлттэй happy path |
| S4 | F1 Valid | T1 Valid | M1 10 USD (=34 000) | C2 USD | L2 Partial | **201 OK** | хэвийн хослол |
| S5 | F3 Missing | T1 Valid | M1 | C1 | — | **200 ERROR_NO_FROM** | [ERROR] |
| S6 | F2 Frozen | T1 Valid | M1 | C1 | L1 | **200 ERROR_FROZEN** | [ERROR] |
| S7 | F1 Valid | T2 Missing | M1 | C1 | L1 | **200 ERROR_NO_TO** | [ERROR] |
| S8 | F1 Valid | T3 SameAsFrom | M1 | C1 | L1 | **200 ERROR_SAME_ACCOUNT** | [ERROR] |
| S9 | F1 Valid | T1 Valid | M2 0 | C1 | L1 | **200 ERROR_BAD_AMOUNT** | [ERROR] |
| S10 | F1 Valid | T1 Valid | M3 −5 000 | C1 | L1 | **200 ERROR_BAD_AMOUNT** | [ERROR] |
| S11 | F1 Valid (B = 200 000) | T1 Valid | M5 200 001 | C1 | L1 | **200 ERROR_INSUFFICIENT** | [ERROR] |
| S12 | F1 Valid (B = 2 000 000) | T1 Valid | M7 400 001 | C1 | L2 (R = 400 000) | **200 ERROR_LIMIT** | [ERROR] |
| S13 | F1 Valid | T1 Valid | 10 000 (> R = 0) | C1 | L3 Full | **200 ERROR_LIMIT** | [ERROR] далд орчин |
| S14 | F1 Valid (B = 200 000) | T1 Valid | M4 200 000 | C1 | L1 | **201 OK** | [SINGLE] үлдэгдэлтэй тэнцүү |
| S15 | F1 Valid (B = 2 000 000) | T1 Valid | M6 400 000 | C1 | L2 (R = 400 000) | **201 OK** | [SINGLE] лимиттэй тэнцүү |
| S16 | F1 Valid | T4 Frozen | M1 | C1 | L1 | **201 OK** (тодруулах!) | [SINGLE] тодорхойлолт чимээгүй |
| S17 | F2 Frozen | T2 Missing | M1 | C1 | L1 | **200 ERROR_FROZEN** | давхар нөхцөл: эрэмбээр FROZEN > NO_TO |
| S18 | F1 Valid (B = 200 000) | T1 Valid | 500 000 (> B, > R) | C1 | L2 (R = 400 000) | **200 ERROR_INSUFFICIENT** | давхар нөхцөл: INSUFFICIENT > LIMIT |
| S19 | F1 Valid (B = 300 000) | T1 Valid | 100 USD (= 340 000 > B) | C2 USD | L1 | **200 ERROR_INSUFFICIENT** | сонирхолтой: тоо нь 100 < B ч хөрвүүлсний дараа B-ээс давна |

19 спецификаци:
- happy path: S1, S3;
- 7 алдааны анги бүр: NO_FROM S5, FROZEN S6, NO_TO S7, SAME_ACCOUNT S8, BAD_AMOUNT S9/S10,
  INSUFFICIENT S11, LIMIT S12/S13;
- SINGLE: S14–S16;
- давхар нөхцөл: S17, S18;
- хөрвүүлэлтийн хил: S19.
