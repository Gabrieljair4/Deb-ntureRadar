# Deb-ntureRadar
é uma ferramenta CLI em Python que calcula o spread de crédito de debêntures frente ao Tesouro IPCA+, converte em basis points e compara com médias históricas por rating (AAA, AA, A, BBB), emitindo alertas automáticos sobre o prêmio de risco. Entrada interativa e validação de dados.
import requests
from datetime import datetime

# ============================================================
# BASES DE REFERÊNCIA
# ============================================================

SPREAD_MEDIO_POR_RATING = {
    "AAA": 50,     # ~0,50% a.a.
    "AA": 120,     # ~1,20% a.a.
    "A": 250,      # ~2,50% a.a.
    "BBB": 500     # ~5,00% a.a.
}

TAXA_PADRAO_TESOURO_IPCA = 6.50
URL_TESOURO_DIRETO = (
    "https://www.tesourodireto.com.br/json/br/com/b3/tesourodireto/service/api/treasurybondsinfo.json"
)


# ============================================================
# FUNÇÕES AUXILIARES DE ENTRADA (COM VALIDAÇÃO)
# ============================================================

def perguntar_texto(mensagem: str, obrigatorio: bool = True, padrao: str = None) -> str:
    """Solicita um texto ao usuário, com opção de valor padrão."""
    while True:
        if padrao:
            entrada = input(f"{mensagem} [{padrao}]: ").strip()
            if not entrada:
                return padrao
        else:
            entrada = input(f"{mensagem}: ").strip()
        if entrada or not obrigatorio:
            return entrada
        print("   ⚠️  Campo obrigatório. Tente novamente.")


def perguntar_float(mensagem: str, minimo: float = 0.0, maximo: float = 100.0) -> float:
    """Solicita um número decimal com validação de faixa."""
    while True:
        entrada = input(f"{mensagem}: ").strip().replace(",", ".")
        try:
            valor = float(entrada)
            if minimo <= valor <= maximo:
                return valor
            print(f"   ⚠️  Valor deve estar entre {minimo} e {maximo}. Tente novamente.")
        except ValueError:
            print("   ⚠️  Valor inválido. Digite um número (ex: 7.5).")


def perguntar_rating() -> str:
    """Solicita o rating com validação contra a base de médias."""
    opcoes = list(SPREAD_MEDIO_POR_RATING.keys())
    print(f"\n   Ratings disponíveis: {', '.join(opcoes)}")
    while True:
        rating = input("   Rating da debênture: ").strip().upper()
        if rating in SPREAD_MEDIO_POR_RATING:
            return rating
        print(f"   ⚠️  Rating '{rating}' não cadastrado. Escolha entre: {', '.join(opcoes)}.")


def perguntar_sim_nao(mensagem: str) -> bool:
    """Pergunta Sim/Não."""
    while True:
        resp = input(f"{mensagem} (s/n): ").strip().lower()
        if resp in ("s", "sim", "y", "yes"):
            return True
        if resp in ("n", "nao", "não", "no"):
            return False
        print("   ⚠️  Responda com 's' ou 'n'.")


# ============================================================
# OBTENÇÃO DO BENCHMARK
# ============================================================

def obter_taxa_tesouro_ipca() -> float:
    """Busca a taxa do Tesouro IPCA+ Principal via API pública."""
    headers = {"User-Agent": "Mozilla/5.0"}
    try:
        resp = requests.get(URL_TESOURO_DIRETO, headers=headers, timeout=10)
        resp.raise_for_status()
        dados = resp.json()
        for titulo in dados.get("response", {}).get("TrsrBdTradgList", []):
            info = titulo.get("TrsrBd", {})
            nome = info.get("nm", "")
            if "Tesouro IPCA+" in nome and "Juros Semestrais" not in nome:
                taxa = info.get("anulInvstmtRate")
                if taxa:
                    return float(taxa)
        print("⚠️  Tesouro IPCA+ não encontrado. Usando taxa padrão.")
    except Exception as e:
        print(f"⚠️  Erro ao acessar a API do Tesouro Direto: {e}")
        print("   Utilizando taxa padrão de fallback.")
    return TAXA_PADRAO_TESOURO_IPCA


# ============================================================
# CÁLCULO E ANÁLISE
# ============================================================

def calcular_spread_bps(taxa_debenture: float, taxa_benchmark: float) -> float:
    return round((taxa_debenture - taxa_benchmark) * 100, 2)


def avaliar_oportunidade(spread_bps: float, rating: str) -> str:
    media = SPREAD_MEDIO_POR_RATING.get(rating.upper())
    if media is None:
        return "Rating sem média cadastrada."
    if spread_bps >= media * 1.20:
        return "✅ GENEROSO (prêmio acima da média histórica)"
    elif spread_bps <= media * 0.80:
        return "⚠️  ABAIXO DA MÉDIA (prêmio comprimido)"
    else:
        return "⚖️  EM LINHA COM A MÉDIA"


# ============================================================
# COLETA INTERATIVA DOS DADOS DA DEBÊNTURE
# ============================================================

def coletar_dados_debenture() -> dict:
    """Pergunta todos os dados da debênture ao usuário."""
    print("\n" + "-" * 60)
    print("📝 CADASTRO DA DEBÊNTURE")
    print("-" * 60)

    codigo    = perguntar_texto("Código da debênture (ex: PETR25)")
    emissor   = perguntar_texto("Nome do emissor (ex: Petrobras)")
    setor     = perguntar_texto("Setor (ex: Petróleo e Gás)", obrigatorio=False, padrao="Não informado")
    rating    = perguntar_rating()
    taxa_deb  = perguntar_float("Taxa da debênture (% a.a., ex: 7.80)")
    vencimento = perguntar_texto("Vencimento (AAAA-MM-DD)", obrigatorio=False, padrao="Não informado")

    return {
        "codigo": codigo,
        "emissor": emissor,
        "setor": setor,
        "rating": rating,
        "taxa_debenture": taxa_deb,
        "vencimento": vencimento,
    }


# ============================================================
# RELATÓRIO
# ============================================================

def exibir_relatorio(debenture: dict, taxa_benchmark: float):
    taxa_deb = debenture["taxa_debenture"]
    rating   = debenture["rating"].upper()
    spread_bps = calcular_spread_bps(taxa_deb, taxa_benchmark)
    media = SPREAD_MEDIO_POR_RATING.get(rating)
    classificacao = avaliar_oportunidade(spread_bps, rating)

    print("\n" + "=" * 60)
    print("📊 RESULTADO DA ANÁLISE DE SPREAD DE CRÉDITO")
    print("=" * 60)
    print(f"Data da análise: {datetime.now().strftime('%d/%m/%Y %H:%M')}")
    print(f"\n🔹 Debênture: {debenture['codigo']} ({debenture['emissor']})")
    print(f"   Setor:       {debenture['setor']}")
    print(f"   Rating:      {rating}")
    print(f"   Vencimento:  {debenture['vencimento']}")
    print(f"\n🔹 Benchmark (Tesouro IPCA+): {taxa_benchmark:.2f}% a.a.")
    print(f"🔹 Taxa da Debênture:         {taxa_deb:.2f}% a.a.")
    print(f"🔹 Spread de Crédito:         {spread_bps:.2f} bps")
    if media is not None:
        print(f"\n📈 Média histórica para rating {rating}: {media} bps")
        print(f"   Desvio em relação à média: {spread_bps - media:+.2f} bps")
        print(f"\n🚨 ALERTA: {classificacao}")
    print("=" * 60)


# ============================================================
# EXECUÇÃO PRINCIPAL
# ============================================================

def main():
    print("=" * 60)
    print("📊 ANALISADOR DE SPREAD DE CRÉDITO (interativo)")
    print("=" * 60)

    # 1) Perguntar se o usuário quer usar a taxa automática do Tesouro ou digitar
    print("\n🔄 Buscando taxa do Tesouro IPCA+ atualizada...")
    taxa_auto = obter_taxa_tesouro_ipca()
    print(f"   Taxa obtida: {taxa_auto:.2f}% a.a.")

    print("\nDeseja usar essa taxa como benchmark (livre de risco)?")
    if perguntar_sim_nao("   Usar taxa automática"):
        taxa_benchmark = taxa_auto
    else:
        taxa_benchmark = perguntar_float("   Informe a taxa do benchmark (% a.a., ex: 6.50)")
    print(f"   ✅ Benchmark definido: {taxa_benchmark:.2f}% a.a.")

    # 2) Loop de análise de debêntures
    while True:
        debenture = coletar_dados_debenture()
        exibir_relatorio(debenture, taxa_benchmark)

        if not perguntar_sim_nao("\nDeseja analisar outra debênture"):
            break

    print("\n👋 Análise encerrada. Até a próxima!")


if __name__ == "__main__":
    main()
