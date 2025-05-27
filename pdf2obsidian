# pdf2obsidian.py
# .envにMISTRAL_API_KEY=で設定して同じディレクトリにおいてください
"""
指定フォルダ内の PDF を mistral-ocr-latest で Markdown(+画像) 化し、
Obsidian Vault の vault/papers/ 以下へ自動配置するスクリプト。
"""

import os
import re
import base64
import shutil
import requests
import yaml
from pathlib import Path
from dotenv import load_dotenv
from tqdm import tqdm
from mistralai import Mistral

# ────────────────────────────────────────────────────────────
INPUT_PDF_DIR = Path("path/to/your/pdf_folder")   # 変換対象 PDF フォルダ
VAULT_DIR     = Path("path/to/your/obsidian_vault")  # Obsidian Vault ルート
PAPERS_DIR    = VAULT_DIR / "papers"               # 保存先フォルダ
OCR_MODEL     = "mistral-ocr-latest"               # mistral-ocr モデル名
HEADERS       = {"User-Agent": "pdf2obsidian (mailto:you@example.com)"}
# ────────────────────────────────────────────────────────────

# slugify 用正規表現
SLUG_RE = re.compile(r"[^A-Za-z0-9]+")
def slugify(text: str, maxlen: int = 50) -> str:
    s = SLUG_RE.sub("_", text).strip("_")
    return s[:maxlen] if len(s) > maxlen else s

# Markdown 中の画像タグからファイル名を抜き出す
IMG_RE = re.compile(r"!\[.*?\]\((img-\d+\.(?:png|jpg|jpeg))\)")

# 環境変数から API キー読み込み
load_dotenv()
ocr_client = Mistral(api_key=os.getenv("MISTRAL_API_KEY"))


def run_mistral_ocr(pdf: Path, out_dir: Path) -> str:
    """mistral-ocr で PDF → Markdown＋画像 変換、Markdown 本文を返す"""
    out_dir.mkdir(parents=True, exist_ok=True)
    b64 = base64.b64encode(pdf.read_bytes()).decode()
    resp = ocr_client.ocr.process(
        model=OCR_MODEL,
        document={"type": "document_url",
                  "document_url": f"data:application/pdf;base64,{b64}"},
        include_image_base64=True
    )

    # ページごとにマーカーを付けて Markdown を結合
    md_pages = []
    for i, page in enumerate(resp.pages or [], start=1):
        content = page.markdown or ""
        md_pages.append(f"<!-- PAGE {i} -->\n\n{content}")
    md = "\n\n".join(md_pages)

    # Markdown に出現する画像名を抜き出し
    img_names = IMG_RE.findall(md)
    images = [img for page in resp.pages or [] for img in page.images or []]
    for idx, img in enumerate(images):
        name = img_names[idx] if idx < len(img_names) else f"img-{idx}.png"
        _, b64data = img.image_base64.split(",", 1)
        (out_dir / name).write_bytes(base64.b64decode(b64data))

    return md


def process_pdf(pdf: Path):
    """単一 PDF の処理：OCR → Markdown 保存 → PDF コピー"""
    slug   = slugify(pdf.stem)
    folder = PAPERS_DIR / slug
    mdfile = folder / f"{slug}.md"

    print(f"→ OCR: {pdf.name}")
    md_body = run_mistral_ocr(pdf, folder)

    # Markdown 書き込み & PDF コピー
    mdfile.write_text(md_body, encoding="utf-8")
    shutil.copy2(pdf, folder / pdf.name)
    print(f"✓ 完了: {pdf.name}")


def main():
    PAPERS_DIR.mkdir(parents=True, exist_ok=True)
    pdf_list = sorted(INPUT_PDF_DIR.glob("*.pdf"))
    for pdf in tqdm(pdf_list, desc="Processing PDFs"):
        try:
            process_pdf(pdf)
        except Exception as e:
            print(f"[ERROR] {pdf.name}: {e}")


if __name__ == "__main__":
    main()
