# Disponibilidade Concreto — Painel de Resultado Mensal e Acumulado

Painel estático (um único arquivo `index.html`, sem backend) com:

- Velocímetro e barras por empresa do mês mais recente, com meta de 93% destacada;
- Gráfico de disponibilidade acumulada (todos os meses);
- Quadro resumo empresa × mês;
- Tabela de detalhe por chave (empresa, regional, SKU) com filtros;
- Upload mensal das 5 bases-fonte (Stockout, PD, Cartas, Expurgo, Consumo), que roda o cálculo **inteiramente no navegador** de quem faz o upload — nenhum arquivo é enviado a um servidor.

---

## 1. Publicar no GitHub Pages

1. Crie um repositório novo (pode ser privado, se só a organização deve acessar — GitHub Pages funciona em repositórios privados nos planos Team/Enterprise; em repositórios públicos, qualquer pessoa com o link acessa).
2. Suba o arquivo `index.html` (deste pacote) para a raiz do repositório — pelo site do GitHub (Add file → Upload files) ou via git:
   ```bash
   git init
   git add index.html
   git commit -m "Painel de disponibilidade"
   git branch -M main
   git remote add origin https://github.com/SUA-ORG/SEU-REPO.git
   git push -u origin main
   ```
3. No repositório: **Settings → Pages**.
4. Em "Build and deployment" → Source: **Deploy from a branch**.
5. Branch: `main`, pasta: `/ (root)` → **Save**.
6. Aguarde 1–2 minutos. O link aparece no topo da mesma página (formato `https://SUA-ORG.github.io/SEU-REPO/`). Compartilhe esse link com a organização.

Qualquer atualização futura: basta subir uma nova versão do `index.html` (substituindo o arquivo) e o GitHub Pages atualiza sozinho em cerca de 1 minuto.

---

## 2. ⚠️ Importante: como funciona o armazenamento fora do Claude

Este painel foi originalmente construído para rodar dentro do Claude.ai, onde existe um recurso de armazenamento compartilhado nativo (`window.storage`) — todos que abrem o artefato veem os mesmos dados.

**Fora do Claude (GitHub Pages, ou qualquer outro host), esse recurso não existe.** O arquivo já vem preparado com um modo alternativo:

- Se `window.storage` não estiver disponível, o painel usa `localStorage` do navegador.
- **`localStorage` é por navegador/dispositivo, não compartilhado entre pessoas.** Ou seja: se alguém usa o botão discreto (⚙ no rodapé) para subir as bases de um novo mês, esse mês fica salvo **só no navegador dessa pessoa** — quem mais acessar o link vai continuar vendo apenas os meses "seed" (Jan–Jul) originais embutidos no arquivo.
- O painel mostra um aviso amarelo no topo avisando disso automaticamente quando detecta que está rodando fora do Claude.

### Como ter um resultado realmente compartilhado entre todos

Como só uma pessoa faz a atualização mensal, o próprio painel já resolve isso — **não é preciso voltar ao Claude todo mês**:

1. Abra o site publicado, clique no ⚙ discreto no rodapé, digite o código de acesso.
2. Suba os 5 arquivos-fonte do mês e clique em **"Processar e salvar"**.
3. Depois de processado, clique em **"⬇ Baixar index.html atualizado"** (mesmo modal) — isso baixa um novo `index.html` com **todos os meses até agora** (os antigos + o que você acabou de calcular) já embutidos como dado fixo do arquivo.
4. Suba esse `index.html` no repositório do GitHub, substituindo o anterior (pelo site do GitHub: Add file → Upload files → arraste o arquivo → Commit; ou via `git add . && git commit -m "Atualização mês X" && git push`).
5. Pronto — qualquer pessoa que abrir o link já vê o novo mês, sem precisar de nada além do link.

O botão "Baixar index.html atualizado" funciona a qualquer momento (não só depois de processar um mês novo) — ele sempre gera um arquivo com o estado atual de tudo que está carregado no seu navegador.

⚠️ Só um detalhe: se você abrir o painel em navegadores/computadores diferentes ao longo dos meses, cada um guarda seu próprio histórico local (`localStorage`) até você gerar e publicar o `index.html`. Recomendo sempre usar o mesmo navegador para as atualizações mensais, ou — mais seguro ainda — sempre partir do `index.html` que já está publicado no GitHub (baixe-o, abra localmente, faça a atualização, gere o novo e suba de novo) para nunca perder histórico.

Se no futuro a organização quiser eliminar até esse passo manual de subir o arquivo (ex.: múltiplas pessoas atualizando), aí sim vale migrar para backend real:
Trocar a camada de armazenamento por um backend simples (ex.: uma planilha do Google Sheets via API, um pequeno banco como Supabase/Firebase, ou uma GitHub Action que grava um arquivo JSON no próprio repositório a cada upload). Isso exige desenvolvimento adicional — fora do escopo deste pacote, mas o código já está isolado num único objeto `storage` (dentro do `<script>` do `index.html`, procure por `const storage =`), o que facilita trocar só essa parte no futuro.

---

## 3. Código de acesso ao upload mensal

O botão discreto (ícone ⚙ no rodapé) pede um código antes de liberar o formulário de upload. Isso **não é uma segurança real** (o código fica visível no código-fonte do HTML para qualquer pessoa técnica) — serve só para não expor o botão de atualização a cliques casuais.

Para trocar o código:
1. Abra `index.html` num editor de texto.
2. Procure por `ADMIN_PASSCODE`.
3. Troque o valor entre aspas.
4. Suba o arquivo atualizado no repositório.

Se a organização precisar de controle de acesso de verdade (só pessoas autorizadas podem ver ou editar), a alternativa é usar um repositório **privado** do GitHub e restringir quem tem acesso a ele, em vez de confiar no código dentro do painel.

---

## 4. Estrutura deste pacote

```
index.html   → o painel completo (HTML + CSS + JS), pronto para publicar
README.md    → este arquivo
```

Não há dependências de build — é um arquivo estático único. As únicas bibliotecas externas usadas (SheetJS para ler Excel e Chart.js para o gráfico acumulado) são carregadas via CDN (`cdnjs.cloudflare.com`) direto no `<head>` do HTML; a máquina de quem acessa precisa de internet para carregá-las.

---

## 5. Metodologia (referência rápida)

- **Necessidade** = MIN(PD, Requisição)
- **Realizado** = Estoque + Consumo + Expurgo
- **Disponibilidade** = Realizado / Necessidade, limitada a 100% por chave (empresa-regional-SKU)
- **Meta do indicador**: 93%
- Chave de junção: empresa–regional–SKU, com regional convertido pela tabela DE-PARA
- Requisição filtrada por `situacao_carta` em [T, A, P, E, D] ou vazio, e por `data_necessidade` no mês de apuração
- Consumo filtrado pela classe de material 54, valores negativos tratados como zero
- Meses Jan–Jul/2026 foram carregados de uma apuração externa (sem recálculo); a partir de Ago/2026 o cálculo passa a ser feito neste painel, a cada upload mensal
