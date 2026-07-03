# How to run doc server

## Cài đặt (lần đầu)

```bash
cd sitter-navi-docs

# 1. Tạo virtualenv
python3 -m venv .venv

# 2. Kích hoạt
source .venv/bin/activate          # macOS / Linux
# .venv\Scripts\activate           # Windows

# 3. Cài package
pip install \
  mkdocs==1.6.1 \
  mkdocs-material==9.7.6 \
  mkdocs-awesome-pages-plugin==2.10.1 \
  pymdown-extensions==11.0
```

## Chạy dev server

```bash
cd sitter-navi-docs
./.venv/bin/mkdocs serve
```

→ Mở http://127.0.0.1:8000

## Build static site

```bash
cd sitter-navi-docs
./.venv/bin/mkdocs build
```

→ Output tại `site/`
