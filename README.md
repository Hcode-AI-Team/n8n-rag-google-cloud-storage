Aqui está a versão aprimorada e corrigida do seu laboratório. Incorporei a arquitetura sequencial (mais segura para produção), adicionei os ajustes essenciais de código e variáveis que faltavam, e incluí uma nota conceitual enriquecedora para os alunos logo no início.

🏗️ Laboratório Guiado: Arquitetura Resiliente de Ingestão com Cloud Storage
Objetivo: Migrar nossa esteira de ingestão de documentos (RAG). Vamos deixar de usar o Google Drive (que exige que o sistema fique "perguntando" o tempo todo se há arquivos novos) e passar a usar um Formulário web que envia o arquivo diretamente para um Storage Profissional (Google Cloud Storage), disparando o processo de IA de forma limpa e segura.

💡 Você Sabia? Conceito de Arquitetura
Embora estejamos construindo uma arquitetura muito superior à anterior, ela tecnicamente é orientada a Webhook (o formulário "empurra" o dado para o n8n). Em uma Arquitetura Orientada a Eventos (Event-Driven) pura na nuvem, o usuário faria o upload direto para o Storage, e o próprio Cloud Storage dispararia um evento (via Pub/Sub) para acordar o n8n. O que faremos hoje é o primeiro passo corporativo: garantir uma esteira de processamento controlada, resiliente e sem desperdício de recursos!

Passo 0: Infraestrutura - Criando seu Bucket no Google Cloud
Antes de construirmos o fluxo, precisamos de um lugar para guardar os arquivos. No mundo corporativo, usamos "Buckets" (baldes) de armazenamento de objetos.

Como todos os grupos estão no mesmo projeto, a organização é vital. O Google Cloud exige que o nome de um bucket seja único em todo o planeta, não apenas no nosso projeto.

Acesse o Google Cloud Console e verifique se o projeto da aula está selecionado no topo da tela.

No menu lateral esquerdo, vá em Cloud Storage > Buckets.

Clique em CRIAR.

Nomeando o Bucket: Para evitar conflitos mundiais, use a nomenclatura estrita: aula-rag-grupo-X-storage (Substitua o X pelo número do seu grupo). Se o nome já existir, adicione a data de hoje no final. Copie este nome, você precisará dele no n8n!

Região: Escolha a opção Region e selecione us-east1 (Carolina do Sul). Por que essa região? Nossos modelos do Vertex AI estão rodando lá. Manter dados e processamento na mesma região zera a latência e corta custos de transferência!

Controle de Acesso: Escolha Uniforme para facilitar as permissões do nosso laboratório.

Deixe as proteções de dados desmarcadas para este exercício e clique em CRIAR.

Passo 1: Limpeza do Fluxo Antigo (Desacoplando o Drive)
Vamos remover a arquitetura antiga baseada em "Polling" (que ficava rodando a cada minuto gastando recursos à toa).

Abra o workflow de ingestão da aula passada no n8n.

Localize e exclua os nós New File in Drive e Download File.

Mantenha no painel o nó Check File State e tudo o que vem depois dele (Extract Text from PDF, Chunk, Embeddings, etc.). Vamos reconectá-los de forma mais inteligente.

Passo 2: A Nova Porta de Entrada (n8n Form Trigger)
Vamos criar uma interface amigável sem escrever nenhuma linha de HTML.

Adicione um novo nó chamado n8n Form Trigger e posicione-o no extremo esquerdo do fluxo.

Abra o nó e configure:

Form Title: "📚 Base de Conhecimento - IA"

Form Description: "Selecione o arquivo PDF para alimentar nossa IA."

Na seção Form Fields, adicione um novo campo:

Field Label: documento (Atenção: escreva exatamente assim em minúsculas. Esta é a nossa variável interna).

Field Type: File

Accept File Types: .pdf

Marque a chave Required Field.

Passo 3: Fazendo o Upload para a Nuvem (GCS API)
Agora, vamos enviar o arquivo recebido para o bucket que criamos.

Adicione um nó HTTP Request logo à direita do Formulário e conecte a saída do Formulário à entrada dele.

Configure o nó HTTP Request:

Method: POST

URL: Copie e cole o endereço abaixo, substituindo NOME-DO-SEU-BUCKET pelo nome que você copiou no Passo 0:
=https://storage.googleapis.com/upload/storage/v1/b/NOME-DO-SEU-BUCKET/o?uploadType=media&name={{ $binary.documento.fileName }}

Authentication: Predefined Credential Type > Google API.

Credential: Selecione a Service Account disponibilizada pelo professor.

Ative a chave Send Body.

Content Type: binaryData

Input Data Field Name: documento

Passo 4: A Esteira Sequencial (Resiliência)
Em produção, se o Google Cloud falhar (ex: erro de permissão), não queremos que a IA gaste recursos processando um PDF que nem foi salvo. Por isso, faremos uma esteira sequencial e segura.

Conecte a saída do HTTP Request à entrada do nó Extract Text from PDF.
(O arquivo binário "documento" viaja pelo fluxo, então o extrator de PDF ainda consegue lê-lo perfeitamente mesmo após o upload ser concluído).

Conecte a saída do Extract Text from PDF na entrada do seu nó antigo Check File State (Firestore).

A ordem lógica agora é: Recebe PDF -> Salva na Nuvem -> Lê o Texto -> Verifica se já existe no Banco.

Passo 5: Refatorando o Estado e Código de IA
Como não temos mais o "ID do Google Drive", atualizaremos o fluxo para usar a "Assinatura Digital" (MD5 Hash) que o Cloud Storage gera automaticamente. Também precisamos avisar ao nosso código JavaScript de onde ler o texto extraído.

Atualizando a Busca no Banco: Abra o nó Check File State. Onde estava o ID do Drive, atualize o Document ID para higienizar o Hash do novo arquivo:

={{ $('HTTP Request').item.json.md5Hash.replace(/\//g, '_') }}

Atualizando a Condição: Abra o nó File Changed? (If). Na Condição 2, altere o Right Value para:

={{ $('HTTP Request').item.json.md5Hash }}

🚨 Crítico: Corrigindo o Código JavaScript: Abra o nó Chunk Text. Como alteramos a estrutura do fluxo, precisamos indicar explicitamente de onde vem o texto. Altere a primeira linha de código para:

const extractedText = $('Extract Text from PDF').first().json.text || '';

Atualizando o ID dos Chunks: Abra o nó Create Deterministic IDs (Set). Altere a expressão do campo deterministic_id para garantir que seja único e sem caracteres inválidos:

={{ $('HTTP Request').item.json.md5Hash.replace(/\//g, '_') }}\_{{ $itemIndex }}

Atualizando o Cadastro do Banco: Abra o nó Prepare State Data (Set) no final do fluxo e atualize as variáveis file_id e hash para:

={{ $('HTTP Request').item.json.md5Hash.replace(/\//g, '_') }}

Passo 6: O Momento da Verdade (Teste Final)
Sempre testamos antes de colocar em produção!

Salve seu Workflow.

Clique no botão inferior Test Workflow. Uma tela com um botão para abrir a URL do formulário vai aparecer.

Acesse a URL, escolha um PDF pequeno (como um currículo ou artigo de 1 a 3 páginas) e clique em enviar.

Volte imediatamente para a tela do n8n. Você verá os dados correndo sequencialmente pelos nós.

Verifique se o upload foi feito no HTTP Request, se o texto foi lido e particionado no Chunk Text e se tudo foi indexado no Vertex AI e Firestore. Se todos os nós ficarem verdes, parabéns! Sua arquitetura corporativa de ingestão RAG está pronta, resiliente e operante.
