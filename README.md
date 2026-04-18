1. Back-end: Coleta e Engenharia de Atributos (coleta.R)
Este script automatiza a coleta de dados de fontes oficiais (como o Sidra/IBGE) e cria os "leads" (antecipações) necessários para o modelo preditivo
.
# Bibliotecas necessárias [4]
library(sidrar)
library(dplyr)
library(lubridate)

# Função para coletar e tratar dados [4, 5]
get_macro_data <- function() {
  # Exemplo: Coleta da Taxa de Desocupação via API Sidra [4]
  raw_desemprego <- sidrar::get_sidra(api = "/t/6381/n1/all/v/4099/p/all/d/v4099%201")
  
  # Tratamento e Engenharia de Atributos [5, 6]
  # Incluindo variáveis de Table 1: volume, cpi, unemp, int_rate e leads [6]
  dados_processados <- raw_desemprego %>%
    dplyr::select(date = "Trimestre Móvel (Código)", value = "Valor") %>%
    dplyr::mutate(
      date = lubridate::ym(date),
      lag_t28 = lag(value, 28),             # Lag de 28 dias [6]
      lead_t28_int_rate = lead(value, 28)    # Lead de 28 dias (proxy de expectativa) [6, 7]
    )
  
  # Salva os dados para o front-end [8]
  save(dados_processados, file = "dados_dashboard.Rdata")
}

get_macro_data()
2. Front-end: Dashboard Interativo (dashboard.Rmd)
Este arquivo utiliza o pacote flexdashboard e Shiny para criar a interface visual dinâmica
.
---
title: "Monitor Macro & Predictor"
output: 
  flexdashboard::flex_dashboard:
    orientation: rows
    social: menu
runtime: shiny
---

```{r setup, include=FALSE}
library(flexdashboard)
library(shiny)
library(ggplot2)
# Carrega os dados processados pelo back-end [11]
load("dados_dashboard.Rdata")
Visão Geral {data-icon="fa-signal"}
Row
Status do Mercado (Otimismo vs Pessimismo)
# Lógica de "termômetro" baseada na teoria macroeconômica [3]
flexdashboard::valueBox("COMPRA", icon = "fa-arrow-up", color = "green")
Taxa Selic Atual
flexdashboard::valueBox("14,75%", icon = "fa-percentage", color = "blue")
Row
Gráfico de Tendência com Leads Macro
shiny::renderPlot({
  ggplot(dados_processados, aes(x = date, y = value)) +
    geom_line(size = 1.5, color = "#282f6b") +
    theme_minimal() +
    labs(title = "Evolução do Indicador vs Expectativa Futura", caption = "Fonte: IBGE/Sidra")
})

### 3. Implementação do Modelo Preditivo (LGBM)
Para integrar a previsão ao aplicativo, você deve treinar um modelo **LGBM** fora do dashboard e carregar apenas o modelo salvo para gerar as predições em tempo real [15, 16].
*   **Vantagem do LGBM:** Ele expande a árvore de forma vertical (*leaf-wise*) e utiliza técnicas como **GOSS** e **EFB** para ser mais eficiente e preciso [15, 17].
*   **Resultados:** A inclusão das variáveis de "lead" (juros futuros de 28 dias) reduz o erro de previsão (**RMSE**) de 1.75 para **1.61** [18, 19].

### 4. Como colocar "no ar" (Implantação)
Para que o aplicativo funcione de forma autônoma e produtiva [1]:
1.  **Hospedagem:** Publique no **ShinyApps.io** ou em um servidor próprio com RStudio Connect.
2.  **Automação:** Use o **Github Actions** para programar a execução do script `coleta.R` diariamente. Isso garante que os dados e as previsões estejam sempre atualizados sem intervenção manual [2].
3.  **Monitoramento:** Utilize o painel para verificar o **viés macro** (otimismo vs. pessimismo) antes de suas operações de day trade [12, 20].

**Aviso:** Informações sobre como configurar especificamente o Github Actions ou servidores de hospedagem dependem de serviços externos às fontes e devem ser verificadas de forma independente.
