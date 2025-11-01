"""
Orçamento de Aluguel - aplicativo CLI (arquivo único)
Funcionalidades implementadas conforme enunciado:
- Tipos: Apartamento (R$700 / 1 quarto), Casa (R$900 / 1 quarto), Estúdio (R$1200)
- Acrescenta valores para 2 quartos, vagas de garagem/estacionamento conforme regras
- Contrato imobiliário: R$2000, parcelável em até 5 vezes
- Desconto de 5% em apartamentos para pessoas sem crianças
- Gera CSV com 12 parcelas do orçamento (arquivo: orcamento_12_parcelas.csv)
- Mostra detalhamento do valor mensal e das parcelas do contrato

Como usar:
- Execute: python orcamento_aluguel.py
- Siga o prompt (o script tenta ser tolerante a entradas)

Observações de implementação (decisões tomadas):
- Por requisito (i)–(iii), preços base definidos como constantes.
- O valor do contrato (R$2000) é dividido pelo número de parcelas escolhido (1 a 5).
- O CSV gerado contém 12 linhas (meses). Nas primeiras `n` linhas constará o valor da parcela do contrato, nas demais, zero.
- O valor mensal do aluguel apresentado inclui o aluguel base + acréscimos - descontos. O contrato aparece separadamente como parcela quando aplicável.

"""
import csv
from decimal import Decimal, ROUND_HALF_UP

# --- constantes ---
PRECO_APARTAMENTO_1Q = Decimal('700.00')
PRECO_CASA_1Q = Decimal('900.00')
PRECO_ESTUDIO = Decimal('1200.00')
CONTRATO_VALOR = Decimal('2000.00')

AD_APART_2Q = Decimal('200.00')
AD_CASA_2Q = Decimal('250.00')
AD_VAGA_GARAGEM = Decimal('300.00')

ESTUDIO_2_VAGAS = Decimal('250.00')
ESTUDIO_VAGA_EXTRA = Decimal('60.00')

DESCONTO_APART_SEM_CRIANCAS = Decimal('0.05')  # 5%

# --- utilitários ---

def money(v: Decimal) -> str:
    """Formata Decimal para string com duas casas e separador de moeda."""
    return f"R$ {v.quantize(Decimal('0.01'), rounding=ROUND_HALF_UP)}"


def parse_int(prompt, allowed=None, default=None):
    while True:
        raw = input(prompt).strip()
        if raw == '' and default is not None:
            return default
        try:
            v = int(raw)
            if allowed and v not in allowed:
                print(f"Entrada inválida. Valores permitidos: {allowed}")
                continue
            return v
        except ValueError:
            print("Por favor digite um número inteiro válido.")


def parse_choice(prompt, choices_map, default=None):
    """choices_map: dict de opção->valor
    retorna a chave selecionada"""
    opts = ", ".join([f"{k}({v})" for k, v in choices_map.items()])
    while True:
        raw = input(f"{prompt} [{opts}] ").strip().lower()
        if raw == '' and default is not None:
            return default
        for k in choices_map:
            if raw == k.lower() or raw == k:
                return k
        print("Opção inválida. Tente novamente.")

# --- cálculo do orçamento ---

def calcular_aluguel(tipo: str, quartos: int, vaga: int, tem_criancas: bool) -> Decimal:
    """Retorna o valor mensal do aluguel (sem considerar as parcelas do contrato)."""
    tipo = tipo.lower()
    valor = Decimal('0.00')
    if tipo == 'apartamento':
        valor = PRECO_APARTAMENTO_1Q
        if quartos == 2:
            valor += AD_APART_2Q
        if vaga >= 1:
            valor += AD_VAGA_GARAGEM
        if (not tem_criancas):
            # desconto de 5% sobre o valor do aluguel (aplicável somente para apartamentos)
            valor = (valor * (Decimal('1.00') - DESCONTO_APART_SEM_CRIANCAS)).quantize(Decimal('0.01'), rounding=ROUND_HALF_UP)
    elif tipo == 'casa':
        valor = PRECO_CASA_1Q
        if quartos == 2:
            valor += AD_CASA_2Q
        if vaga >= 1:
            valor += AD_VAGA_GARAGEM
    elif tipo == 'estudio':
        valor = PRECO_ESTUDIO
        # vaga para estudio: desconto/valor específico
        # regra: pode ser adicionado vagas de estacionamento no valor de R$250,00 com 2 vagas,
        # podendo acrescentar mais vagas no valor de R$60,00 cada
        if vaga >= 2:
            # os primeiros 2 custam R$250 (se o usuario informar 1 vaga, assumimos que é 2 para encaixe da regra?)
            # iremos cobrar R$250 quando vaga >= 2 para cobrir 2 vagas, e 60 por vaga extra além de 2
            valor += ESTUDIO_2_VAGAS
            extra = max(0, vaga - 2)
            valor += ESTUDIO_VAGA_EXTRA * extra
        elif vaga == 1:
            # interpretação: 1 vaga -> cobrar proporcional? Para sermos conservadores cobraremos R$250 por até 2 vagas.
            valor += ESTUDIO_2_VAGAS
    else:
        raise ValueError('Tipo de imóvel desconhecido')

    return valor.quantize(Decimal('0.01'), rounding=ROUND_HALF_UP)


def gerar_csv_parcelas(valor_aluguel: Decimal, contrato_valor: Decimal, parcelas_contrato: int, nome_arquivo: str = 'orcamento_12_parcelas.csv'):
    """Gera CSV com 12 meses e as parcelas do contrato aplicadas nos primeiros N meses.
    Colunas: Mes, Valor_Aluguel, Parcela_Contrato, Total_Mensal
    """
    parcelas_contrato = max(1, min(5, parcelas_contrato))
    parcela_valor = (contrato_valor / parcelas_contrato).quantize(Decimal('0.01'), rounding=ROUND_HALF_UP)

    with open(nome_arquivo, mode='w', newline='', encoding='utf-8') as f:
        writer = csv.writer(f)
        writer.writerow(['Mes', 'Valor_Aluguel', 'Parcela_Contrato', 'Total_Mensal'])
        for mes in range(1, 13):
            parc = parcela_valor if mes <= parcelas_contrato else Decimal('0.00')
            total = (valor_aluguel + parc).quantize(Decimal('0.01'), rounding=ROUND_HALF_UP)
            writer.writerow([mes, f"{valor_aluguel}", f"{parc}", f"{total}"])

    return nome_arquivo


# --- interface simples ---

def main():
    print('=== Gerador de Orçamento - R.M Imobiliária ===')
    tipo_map = {'Apartamento': 'Apartamento', 'Casa': 'Casa', 'Estudio': 'Estudio'}
    tipo = parse_choice('Escolha o tipo de imóvel', tipo_map, default='Apartamento')

    quartos = 1
    if tipo.lower() in ('apartamento', 'casa'):
        quartos = parse_int('Quantos quartos? (1 ou 2) [padrão 1]: ', allowed=[1, 2], default=1)
    else:
        # estudio -> quartos fixos (estúdio não tem 2 quartos) mas permitimos vaga
        quartos = 1

    vaga = 0
    if tipo.lower() in ('apartamento', 'casa'):
        vaga = parse_int('Deseja vaga de garagem? Quantas vagas (0 para não): ', default=0)
    else:  # estudio
        vaga = parse_int('Quantas vagas deseja para o estúdio? (0, 1, 2, ...): ', default=0)

    tem_criancas = True
    if tipo.lower() == 'apartamento':
        raw = input('Há crianças no núcleo familiar? (s/n) [padrão s]: ').strip().lower()
        if raw == '' or raw.startswith('s'):
            tem_criancas = True
        else:
            tem_criancas = False

    parcelas_contrato = parse_int('Deseja parcelar o contrato de R$2000 em quantas vezes? (1 a 5) [padrão 5]: ', allowed=[1,2,3,4,5], default=5)

    valor_aluguel = calcular_aluguel(tipo, quartos, vaga, tem_criancas)
    parcela_valor = (CONTRATO_VALOR / parcelas_contrato).quantize(Decimal('0.01'), rounding=ROUND_HALF_UP)

    print('\n--- Resumo do Orçamento ---')
    print(f'Tipo: {tipo}')
    print(f'Quartos: {quartos}')
    print(f'Vagas: {vaga}')
    print(f'Valor do aluguel (mensal): {money(valor_aluguel)}')
    print(f'Valor do contrato: {money(CONTRATO_VALOR)} parcelado em {parcelas_contrato}x -> parcela: {money(parcela_valor)}')
    print(f'Valor total (aluguel + 1ª parcela do contrato): {money((valor_aluguel + parcela_valor).quantize(Decimal("0.01"), rounding=ROUND_HALF_UP))}')

    # gerar CSV com 12 parcelas do orçamento
    nome_csv = gerar_csv_parcelas(valor_aluguel, CONTRATO_VALOR, parcelas_contrato)
    print(f'CSV de 12 parcelas gerado: {nome_csv} (colunas: Mes, Valor_Aluguel, Parcela_Contrato, Total_Mensal)')

    print('\nObrigado! Utilize o CSV para anexar ao orçamento mensal do cliente.')


if __name__ == '__main__':
    main()
