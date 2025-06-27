import os
import base64
from fastapi import FastAPI, File, UploadFile, HTTPException, Form
from fastapi.responses import JSONResponse, Response
import requests
from dotenv import load_dotenv

# Carrega as variáveis de ambiente do arquivo .env
# Certifique-se de ter um arquivo .env na raiz do projeto com:
# STABILITY_API_KEY="SUA_CHAVE_DE_API_DA_STABILITY_AI_AQUI"
load_dotenv()

# Inicializa o FastAPI, que é o seu servidor web para a API
app = FastAPI(title="DecoraIA Backend MVP")

# Obtém a chave da API da Stability AI das variáveis de ambiente
# É crucial que esta chave esteja configurada para que a API funcione.
STABILITY_API_KEY = os.getenv("STABILITY_API_KEY")
if not STABILITY_API_KEY:
    raise ValueError("STABILITY_API_KEY não está configurada no arquivo .env. Crie-o!")

# ID do motor de geração da Stability AI.
# Você pode verificar os IDs mais recentes e adequados na documentação da Stability AI.
# Exemplos comuns: "stable-diffusion-v1-6", "stable-diffusion-xl-1024-v1-0".
# Recomendo usar um modelo SDXL para melhor qualidade e fotorrealismo.
STABILITY_ENGINE_ID = "stable-diffusion-xl-1024-v1-0" # Exemplo: Usando SDXL para melhor qualidade
STABILITY_API_URL = f"https://api.stability.ai/v1/generation/{STABILITY_ENGINE_ID}/image-to-image"

# --- Lógica para Construir o Prompt da IA ---
# Esta função pega as escolhas do usuário e as expande em um prompt detalhado para a IA.
# É aqui que o conhecimento do seu "deep search" é aplicado.
def build_ai_prompt(request_data: dict) -> str:
    """
    Constrói um prompt detalhado para a IA, combinando as escolhas do usuário
    com o conhecimento das tendências e estilos do seu relatório.
    """
    room_type = request_data.get("room_type", "room").lower()
    style = request_data.get("style", "modern").lower()
    color_palette = request_data.get("color_palette", "natural and earthy tones").lower()
    room_size = request_data.get("room_size", "medium").lower()

    # Prompt base, incluindo o tamanho do cômodo para melhor adaptação
    base_prompt = f"A photorealistic interior design of a {room_size} {room_type}, " \
                  f"embodying the **{style} style**."
    
    # Adicionar detalhes de cores com base nas suas categorias do relatório
    if "naturais e terrosos" in color_palette:
        base_prompt += " The dominant color palette is **natural and earthy tones** like light beige, warm off-white, olive green, and terracotta, creating a cozy and comforting atmosphere. Combined with sustainable materials."
    elif "vibrantes e energéticas" in color_palette:
        base_prompt += " The primary colors are **vibrant and energetic**, with pops of bold red, mustard yellow, turquoise, and emerald green integrated through accent furniture, artwork, or decorative objects, evoking joy and self-expression."
    elif "neutros sofisticados e calmantes" in color_palette:
        base_prompt += " The dominant color palette consists of **sophisticated and calming neutrals** such as light gray, subtle dusty rose, off-white, and deep tranquil blues, promoting tranquility and elegance."
    
    # Adicionar detalhes do estilo (simplificado para MVP, mas extensível com o seu relatório)
    if "japandi" in style:
        base_prompt += " It features a harmonious blend of Japanese minimalism and Scandinavian functionality. Include minimalist furniture in light wood tones, linen upholstery, natural textures, woven rugs, and strategically placed indoor plants. The ambiance is serene and balanced with soft, natural light, emphasizing clean lines and natural materials."
    elif "industrial" in style:
        base_prompt += " Exposed brick walls, concrete textures, and prominent black metal accents (e.g., shelving, light fixtures). Furniture is robust with a mix of vintage and modern pieces, often made of raw materials. Emphasize dramatic, cinematic lighting that highlights textures and a high ceiling feel. The overall feeling is edgy, creative, and urban."
    elif "soft minimalism" in style:
        base_prompt += " The space should feel uncluttered and airy, focusing on simplicity and comfort. Feature minimalist furniture with clean lines, warm, inviting textiles like soft wool rugs and textured curtains. Natural light is soft and diffused, creating a tranquil atmosphere. Emphasize subtle textures and a sense of peaceful quietude, with invisible storage solutions."
    elif "boho" in style:
        base_prompt += " The space is eclectic and vibrant, mixing diverse patterns and many handcrafted elements. Include ample natural plants, comfortable, rounded furniture (like rattan or wicker), and a variety of colorful cushions, tapestries, and ethnic patterns. The atmosphere is relaxed, free-spirited, and cozy."
    elif "clássico" in style:
        base_prompt += " The design exudes elegance and timelessness, with strong wooden furniture (often dark wood), possibly marble or ornate details, and rich, deep colors (like emerald green or deep burgundy). Include luxurious textures in upholstery, grand light fixtures, and sophisticated decorative elements. The ambiance is formal yet inviting."
    
    # Termos para garantir fotorrealismo e qualidade (Negative Prompt é igualmente importante)
    base_prompt += " High-quality render, architectural photography, hyperrealistic, professional interior photo, coherent lighting, fine details."
    
    return base_prompt

# --- Endpoint da API para Gerar Design ---
# Este endpoint aceita uma imagem (UploadFile) e dados de formulário (Form) para as preferências.
@app.post("/generate_design/")
async def generate_design(
    image: UploadFile = File(...),         # A foto do cômodo enviada pelo usuário
    room_type: str = Form(...),            # Tipo do cômodo (ex: "sala de estar")
    style: str = Form(...),                # Estilo de decoração (ex: "Japandi")
    color_palette: str = Form(...),        # Paleta de cores (ex: "Tons Naturais e Terrosos")
    room_size: str = Form("medium")        # Tamanho do cômodo (ex: "small", "medium", "large")
):
    """
    Endpoint para gerar um design de interiores com IA, usando uma foto do cômodo como base.
    
    Args:
        image (UploadFile): A foto do cômodo que o usuário quer decorar.
        room_type (str): O tipo de cômodo (ex: "sala de estar", "quarto").
        style (str): O estilo de decoração desejado (ex: "Japandi", "Industrial").
        color_palette (str): A paleta de cores principal (ex: "Tons Naturais e Terrosos").
        room_size (str, opcional): O tamanho do cômodo (ex: "small", "medium", "large"). Padrão é "medium".
        
    Returns:
        JSONResponse: Um JSON contendo a imagem gerada em Base64.
        HTTPException: Erro se a comunicação com a IA falhar ou se a imagem não for gerada.
    """
    
    # 1. Lê a imagem enviada pelo usuário e a codifica para Base64
    try:
        image_bytes = await image.read()
        image_base64 = base64.b64encode(image_bytes).decode("utf-8")
    except Exception as e:
        raise HTTPException(status_code=400, detail=f"Não foi possível ler a imagem enviada: {e}")

    # Coleta os dados da requisição para construir o prompt
    request_data_dict = {
        "room_type": room_type,
        "style": style,
        "color_palette": color_palette,
        "room_size": room_size
    }

    # 2. Constrói o prompt detalhado para a IA da Stability AI
    ai_prompt = build_ai_prompt(request_data_dict)
    print(f"Prompt da IA gerado: {ai_prompt}") # Para fins de depuração no terminal

    # Configura os cabeçalhos da requisição HTTP
    headers = {
        "Accept": "application/json",
        "Authorization": f"Bearer {STABILITY_API_KEY}"
    }

    # 3. Prepara o payload para a API da Stability AI (Image-to-Image)
    # A API image-to-image da Stability AI espera a imagem original (init_image) em base64
    # e os prompts de texto.
    payload = {
        "init_image": image_base64,
        "init_image_mode": "IMAGE_STRENGTH", # Modo de influência da imagem inicial
        "image_strength": 0.35, # Força da imagem inicial (0.0 a 1.0). 0.35 permite bastante criatividade da IA.
                                # Valores maiores (ex: 0.7-0.9) mantêm mais da estrutura original da foto.
        "text_prompts": [
            {"text": ai_prompt, "weight": 1},
            # Negative prompt para remover elementos indesejados
            {"text": "ugly, distorted, blurry, low resolution, bad anatomy, deformed, disfigured, poor lighting, unrealistic, cartoon, sketch, painting, text, watermarks, bad composition", "weight": -1}
        ],
        "cfg_scale": 7,     # Escala de guia de prompt (como a IA segue o prompt). Experimente 7-12.
        "height": 768,      # Altura da imagem gerada. Múltiplos de 64, ex: 512, 768, 1024.
        "width": 768,       # Largura da imagem gerada. Múltiplos de 64.
        "samples": 1,       # Número de imagens a gerar por requisição. Para MVP, comece com 1.
        "steps": 30,        # Número de passos de difusão. Experimente 30-50 para melhor qualidade.
        "seed": 0           # Seed para resultados reproduzíveis (0 para resultados aleatórios).
    }

    try:
        # Faz a requisição POST para a API da Stability AI
        # O cabeçalho Content-Type é automaticamente definido como application/json pelo requests.post
        response = requests.post(STABILITY_API_URL, headers=headers, json=payload)
        response.raise_for_status() # Lança um erro para status de resposta HTTP 4xx/5xx

        api_response_data = response.json()
        
        # Verifica se a IA retornou imagens
        if not api_response_data.get("artifacts"):
            raise HTTPException(status_code=500, detail="A Stability AI não retornou nenhuma imagem.")
        
        # Pega a primeira imagem gerada (em Base64)
        image_b64 = api_response_data["artifacts"][0]["base64"]
        
        # Retorna a imagem em Base64 como parte da resposta JSON
        return JSONResponse(content={"image_base64": image_b64})

    except requests.exceptions.HTTPError as http_err:
        # Captura erros HTTP específicos da API da Stability AI
        print(f"Erro HTTP da Stability AI: {http_err.response.status_code} - {http_err.response.text}")
        raise HTTPException(status_code=http_err.response.status_code, detail=f"Erro da API da Stability AI: {http_err.response.text}")
    except requests.exceptions.RequestException as req_err:
        # Captura erros gerais de requisição (rede, timeout, etc.)
        raise HTTPException(status_code=500, detail=f"Erro de comunicação com a API da Stability AI: {req_err}")
    except Exception as e:
        # Captura quaisquer outros erros inesperados
        print(f"Erro interno no servidor: {e}")
        raise HTTPException(status_code=500, detail=f"Erro interno ao processar sua requisição: {e}")

# --- Endpoint Raiz (Opcional, para testar se a API está no ar) ---
@app.get("/")
async def read_root():
    """
    Endpoint de teste para verificar se o backend está funcionando.
    """
    return {"message": "Bem-vindo ao DecoraIA Backend! Acesse /docs para testar o endpoint de geração de design."}

# --- Instruções para Rodar o Backend Localmente ---
# 1. Certifique-se de ter Python 3.9+ instalado.
# 2. Crie uma pasta para o seu projeto.
# 3. Dentro da pasta, crie um arquivo chamado `requirements.txt` com o conteúdo:
#    fastapi[all]
#    python-dotenv
#    requests
# 4. Abra o terminal na pasta do projeto e instale as dependências:
#    pip install -r requirements.txt
# 5. Crie um arquivo `.env` na raiz da pasta do projeto e adicione sua chave de API:
#    STABILITY_API_KEY="SUA_CHAVE_DE_API_DA_STABILITY_AI_AQUI"
#    (Substitua pelo valor real da sua chave do DreamStudio)
# 6. Salve o código acima em um arquivo chamado `main.py` (ou `app.py`) dentro da pasta do projeto.
# 7. No terminal, na mesma pasta, execute o comando para iniciar o servidor:
#    uvicorn main:app --reload
# 8. Abra seu navegador e acesse: http://127.0.0.1:8000/docs
#    Você verá a documentação interativa da sua API (Swagger UI).
# 9. No Swagger UI, expanda o endpoint `/generate_design/` e clique em "Try it out".
#    Você poderá:
#    - Fazer upload de uma imagem do seu computador.
#    - Preencher os campos `room_type`, `style`, `color_palette` e `room_size` (usando as opções que definimos no MVP).
#    - Clique em "Execute" para testar a geração de design.
#    A resposta conterá a imagem gerada em formato Base64.
