from decimal import Decimal, ROUND_HALF_UP
import csv
from pathlib import Path

class OrcamentoAluguel:
    def __init__(self, tipo, quartos, vagas, tem_criancas, parcelas_contrato=5):
        self.tipo = tipo.lower()
        self.quartos = quartos
        self.vagas = vagas
        self.tem_criancas = tem_criancas
        self.parcelas_contrato = max(1, min(5, parcelas_contrato))
        self.valor_contrato = Decimal('2000.00')
        self.valor_aluguel = self.calcular_aluguel()

    def calcular_aluguel(self):
        valor = Decimal('0.00')

        if self.tipo == 'apartamento':
            valor = Decimal('700.00')
            if self.quartos == 2:
                valor += Decimal('200.00')
            if self.vagas >= 1:
                valor += Decimal('300.00')
            if not self.tem_criancas:
                valor *= Decimal('0.95')  # desconto 5%

        elif self.tipo == 'casa':
            valor = Decimal('900.00')
            if self.quartos == 2:
                valor += Decimal('250.00')
            if self.vagas >= 1:
                valor += Decimal('300.00')

        elif self.tipo == 'estudio':
            valor = Decimal('1200.00')
            if self.vagas >= 2:
                valor += Decimal('250.00')
                extra = max(0, self.vagas - 2)
                valor += Decimal('60.00') * extra
            elif self.vagas == 1:
                valor += Decimal('125.00')  # metade de 2 vagas

        return valor.quantize(Decimal('0.01'), rounding=ROUND_HALF_UP)

    def gerar_csv(self, caminho_arquivo):
        parcela = (self.valor_contrato / self.parcelas_contrato).quantize(Decimal('0.01'))
        path = Path(caminho_arquivo)

        with path.open('w', newline='', encoding='utf-8') as f:
            writer = csv.writer(f)
            writer.writerow(['Mês', 'Valor_Aluguel', 'Parcela_Contrato', 'Total_Mensal'])
            for mes in range(1, 13):
                parc = parcela if mes <= self.parcelas_contrato else Decimal('0.00')
                total = (self.valor_aluguel + parc).quantize(Decimal('0.01'))
                writer.writerow([mes, f"R$ {self.valor_aluguel}", f"R$ {parc}", f"R$ {total}"])

        return str(path)

    def resumo(self):
        print(f"Tipo: {self.tipo.capitalize()}")
        print(f"Quartos: {self.quartos}")
        print(f"Vagas: {self.vagas}")
        print(f"Crianças: {'Sim' if self.tem_criancas else 'Não'}")
        print(f"Valor do aluguel: R$ {self.valor_aluguel}")
        print(f"Contrato: R$ {self.valor_contrato} / {self.parcelas_contrato}x")


# Exemplo de uso
if __name__ == "__main__":
    aluguel = OrcamentoAluguel("Apartamento", 2, 1, False, parcelas_contrato=5)
    aluguel.resumo()
    arquivo = aluguel.gerar_csv("../output/orcamento_12_parcelas.csv")
    print(f"\nArquivo gerado: {arquivo}")
