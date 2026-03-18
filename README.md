# Automação: Google Meet para YouTube com Resumable Upload

## Sobre o Projeto
Este script automatiza o envio de gravações longas do Google Meet (salvas no Google Drive) diretamente para uma playlist específica no YouTube. O grande diferencial deste código é a implementação de *Resumable Uploads* (upload em fatias). 

Isso resolve dois problemas clássicos do Google Apps Script:
- O limite de memória temporária para arquivos maiores que 50 MB.
- O limite de tempo máximo de execução contínua de 6 minutos.

## O que o script faz
- Monitora a pasta do Drive e identifica apenas arquivos de vídeo MP4, ignorando documentos de texto e listas de presença.
- Limpa o nome padrão gerado pelo Meet e padroniza o título para o YouTube no formato: Título da Aula - RX - DD-MM-AAAA.
- Fatiamento de dados: Pega o vídeo no Drive e envia para o YouTube em pedaços de 5 MB.
- Relógio interno: Aos 4 minutos de execução, o script pausa o trabalho, salva o progresso na memória e cria um alarme invisível para acordar 1 minuto depois e continuar de onde parou.
- Ao finalizar os 100% do envio, insere o vídeo como Não Listado em uma playlist e move o arquivo original para uma pasta de processados no Drive.



## Pré-requisitos
Para configurar esta automação, você vai precisar de:
- Conta Google com canal do YouTube ativo.
- IDs das pastas do Google Drive (Origem e Destino).
- ID da Playlist do YouTube.

## Como configurar no Google Apps Script

### Fase 1: Coletando as informações necessárias

Antes de ir para o código, você precisa separar três informações cruciais. Deixe-as anotadas em um bloco de notas:

1. ID da pasta Meet Recordings: Vá ao seu Google Drive, abra a pasta onde o Meet salva as gravações e olhe a barra de endereços do seu navegador. Copie apenas o código de letras e números que aparece logo depois de "folders/".

2. ID da pasta de processados: Crie uma pasta no seu Drive para receber os vídeos que já foram enviados (ex: "Enviados para o YouTube"). Abra essa pasta e copie o ID dela na barra de endereços, exatamente igual no passo anterior.

3. ID da Playlist do YouTube: Abra a playlist desejada no YouTube pelo seu navegador. Na barra de endereços, copie o código que aparece logo depois de "list=". Copie apenas a sequência que começa com "PL" e termina antes de qualquer barra ou outro símbolo extra.


### Fase 2: Criando o projeto e ativando as APIs

1. Acesse o site script.google.com e faça login com a mesma conta que contém as gravações e o canal do YouTube.
2. Clique no botão "Novo projeto" no canto superior esquerdo.
3. Dê um nome para o seu projeto clicando em "Projeto sem título" na parte superior.
4. No menu lateral esquerdo, procure pela seção "Serviços" e clique no ícone de adição (+).
5. Na janela que se abrir, role para baixo, selecione "YouTube Data API v3" e clique em "Adicionar".
6. Clique novamente no ícone de adição (+) em "Serviços", selecione "Drive API" e clique em "Adicionar".

### Fase 3: Adicionando e personalizando o código

1. Apague qualquer código que já esteja na tela principal (geralmente aparece um bloco com "function myFunction").
2. Cole o código completo de Resumable Upload (que estará no arquivo principal deste repositório).
3. Nas primeiras linhas do código, procure pelas variáveis idPastaOrigem, idPastaDestino e idPlaylist.
4. Substitua os textos em maiúsculo pelos IDs reais que você coletou na Fase 1. Mantenha as aspas simples ao redor dos IDs. O formato deve ficar parecido com isto: var idPlaylist = 'PLx0...';
5. Clique no ícone de disquete na parte superior para salvar o projeto.

### Fase 4: Autorizando o script na sua conta

1. Na barra superior, verifique se a função selecionada na caixa suspensa é "gerenciarUpload" e clique no botão "Executar".
2. Como é a primeira vez rodando, o Google pedirá que você revise as permissões.
3. Clique em "Revisar permissões" e escolha sua conta Google.
4. Se aparecer um aviso de segurança dizendo que o app não foi verificado, clique em "Avançado", depois em "Acessar projeto (não seguro)" e, por fim, em "Permitir".
5. Deixe o script rodar até o fim ou até ele criar a pausa estratégica de 4 minutos no registro de execução.

### Fase 5: Automatizando a execução (O Gatilho)

Para que o script rode sozinho nos bastidores e vigie sua pasta sem que você precise apertar nenhum botão:

1. No menu lateral esquerdo do Apps Script, clique no ícone de relógio, chamado "Acionadores".
2. No canto inferior direito, clique no botão azul "Adicionar acionador".
3. Configure a janela exatamente com estas opções:
- Escolha a função que será executada: gerenciarUpload
- Escolha a implantação que deve ser executada: Teste (ou Head)
- Selecione a origem do evento: Baseado no tempo
- Selecione o tipo de acionador com base no tempo: Temporizador de horas
- Selecione um intervalo de horas: A cada hora.
4. Clique em "Salvar".
