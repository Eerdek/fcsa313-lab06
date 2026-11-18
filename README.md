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
