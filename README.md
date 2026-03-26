import requests
import json
import base64
from datetime import datetime

# ── Configure aqui ──────────────────────────────────────────
GITHUB_TOKEN = "SEU_TOKEN_AQUI"   # github.com → Settings → Developer settings → Tokens (classic) → repo
REPO_OWNER   = "luizmpjr-code"
REPO_NAME    = "lm-autosolutions"
FILE_PATH    = "estoque.json"
# ────────────────────────────────────────────────────────────

def commit_estoque(json_path="estoque.json"):
    with open(json_path, "r", encoding="utf-8") as f:
        conteudo = f.read()

    conteudo_b64 = base64.b64encode(conteudo.encode()).decode()

    url = f"https://api.github.com/repos/{REPO_OWNER}/{REPO_NAME}/contents/{FILE_PATH}"
    headers = {
        "Authorization": f"token {GITHUB_TOKEN}",
        "Accept": "application/vnd.github.v3+json"
    }

    # Verifica SHA do arquivo existente
    r = requests.get(url, headers=headers)
    sha = r.json().get("sha") if r.status_code == 200 else None

    payload = {
        "message": f"Estoque atualizado — {datetime.now().strftime('%d/%m/%Y %H:%M')}",
        "content": conteudo_b64,
    }
    if sha:
        payload["sha"] = sha

    resp = requests.put(url, headers=headers, json=payload)
    if resp.status_code in [200, 201]:
        print("✓ estoque.json enviado ao GitHub — Netlify fará deploy automático")
    else:
        print(f"Erro GitHub: {resp.status_code} — {resp.text[:300]}"){
  "atualizado": "aguardando primeira sincronização",
  "total": 0,
  "veiculos": []
}# LM AutoSolutions

Site oficial da LM AutoSolutions — especialistas em compra, venda e consultoria automotiva desde 1991.

**URL:** https://lmautosolutions.tech

## Estrutura

```
lm-autosolutions/
├── index.html          # Site principal
├── estoque.json        # Estoque atualizado automaticamente pelo Open Claw
├── scraper_olx.py      # Raspa anúncios do perfil OLX
├── commit_github.py    # Envia estoque.json para o GitHub
├── run_all.py          # Executa scraper + commit (agendar no Open Claw)
└── README.md
```

## Atualização automática do estoque

O Open Claw roda `run_all.py` a cada 6 horas:
1. Raspa os anúncios ativos do perfil OLX da LM AutoSolutions
2. Salva em `estoque.json`
3. Faz commit no GitHub
4. Netlify detecta o commit e faz deploy automático em ~30 segundos

## Configuração

1. Instalar dependências: `pip install requests beautifulsoup4`
2. Gerar token GitHub em: Settings → Developer settings → Personal access tokens → repo
3. Colar token em `commit_github.py`
4. Agendar no Open Claw: `0 */6 * * * python /caminho/run_all.py`

## Contato

WhatsApp: (21) 96811-1066  
Instagram: @lm_autosolutions  
OLX: https://www.olx.com.br/perfil/luiz-miguel-autosolutions-18546a8cfrom scraper_olx import scrape_olx, salvar_json
from commit_github import commit_estoque

print("=" * 50)
print("LM AutoSolutions — Sincronização de estoque")
print("=" * 50)

veiculos = scrape_olx()

if veiculos:
    salvar_json(veiculos)
    commit_estoque()
    print(f"\n✅ {len(veiculos)} veículos sincronizados com o site")
else:
    print("\n⚠ Nenhum veículo encontrado — site mantém estoque anterior")import requests
from bs4 import BeautifulSoup
import json
from datetime import datetime

PERFIL_URL = "https://www.olx.com.br/perfil/luiz-miguel-autosolutions-18546a8c"

HEADERS = {
    "User-Agent": "Mozilla/5.0 (Windows NT 10.0; Win64; x64) "
                  "AppleWebKit/537.36 (KHTML, like Gecko) "
                  "Chrome/120.0.0.0 Safari/537.36",
    "Accept-Language": "pt-BR,pt;q=0.9",
}

def scrape_olx():
    veiculos = []
    try:
        r = requests.get(PERFIL_URL, headers=HEADERS, timeout=15)
        soup = BeautifulSoup(r.text, "html.parser")

        cards = soup.select("li[data-lurker-detail='list_id']")
        if not cards:
            cards = soup.select("ul.sc-1fcmfeb-2 li")

        for card in cards:
            try:
                titulo = card.select_one("h2, .fnmrjs-0, [class*='title']")
                titulo = titulo.get_text(strip=True) if titulo else ""

                preco = card.select_one("p[class*='price'], .sc-ifAKCX, [class*='price']")
                preco = preco.get_text(strip=True) if preco else "Consulte"

                link = card.select_one("a")
                link = link["href"] if link and link.get("href") else PERFIL_URL

                img = card.select_one("img")
                img_url = ""
                if img:
                    img_url = img.get("src") or img.get("data-src") or ""

                detalhes = card.select("[class*='detail'], [class*='property']")
                det_texto = " · ".join([d.get_text(strip=True) for d in detalhes[:3]])

                if titulo:
                    veiculos.append({
                        "titulo": titulo,
                        "preco": preco,
                        "detalhes": det_texto,
                        "link": link if link.startswith("http") else "https://www.olx.com.br" + link,
                        "imagem": img_url,
                        "atualizado": datetime.now().strftime("%d/%m/%Y %H:%M")
                    })
            except Exception as e:
                print(f"Erro no card: {e}")
                continue

    except Exception as e:
        print(f"Erro ao acessar OLX: {e}")

    return veiculos

def salvar_json(veiculos, path="estoque.json"):
    dados = {
        "atualizado": datetime.now().strftime("%d/%m/%Y às %H:%M"),
        "total": len(veiculos),
        "veiculos": veiculos
    }
    with open(path, "w", encoding="utf-8") as f:
        json.dump(dados, f, ensure_ascii=False, indent=2)
    print(f"✓ {len(veiculos)} veículos salvos em {path}")

if __name__ == "__main__":
    print("Raspando OLX...")
    veiculos = scrape_olx()
    salvar_json(veiculos)
    

if __name__ == "__main__":
    commit_estoque()
    
