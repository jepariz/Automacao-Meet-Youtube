# Automação: Google Meet para YouTube com Resumable Upload

## Sobre o Projeto
Este script automatiza o envio de gravações longas do Google Meet (salvas no Google Drive) diretamente para uma playlist específica no YouTube. O grande diferencial deste código é a implementação de uploads em fatias (Resumable Uploads). 

Isso resolve dois problemas clássicos do ecossistema gratuito do Google Apps Script:
- O limite de memória temporária para arquivos maiores que 50 MB.
- O limite de tempo máximo de execução contínua de 6 minutos.

## O que o script faz
- Monitora a pasta do Drive e identifica apenas arquivos de vídeo MP4 (ignora listas de presença e documentos).
- Limpa o nome padrão do Meet e padroniza o título para o formato: Título da Aula - RX - DD-MM-AAAA.
- Fatiamento: Pega o vídeo no Drive e envia para o YouTube em pedaços de 5 MB.
- Relógio interno: Aos 4 minutos de execução, o script pausa o trabalho, salva o progresso e cria um alarme para acordar 1 minuto depois e continuar de onde parou.
- Ao finalizar os 100%, insere o vídeo como Não Listado na playlist e move o arquivo original para uma pasta de processados.

## Passo 1: Coletando as informações necessárias
Antes de ir para o código, separe três IDs cruciais:

1. ID da pasta Meet Recordings: Vá ao seu Google Drive, abra a pasta das gravações e olhe a barra de endereços do navegador. Copie apenas o código que aparece depois de "folders/".
2. ID da pasta de processados: Crie uma pasta no Drive para os vídeos já enviados. Copie o ID dela na barra de endereços, igual ao passo anterior.
3. ID da Playlist do YouTube: Abra a playlist no YouTube. Na barra de endereços, copie o código logo depois de "list=" (começa com PL).

## Passo 2: Criando o projeto e ativando as APIs
1. Acesse script.google.com e faça login.
2. Clique em Novo projeto.
3. No menu lateral esquerdo, em Serviços, clique no ícone de adição (+).
4. Selecione YouTube Data API v3 e clique em Adicionar.
5. Clique novamente no (+), selecione Drive API e clique em Adicionar.

## Passo 3: O Código
Apague o código padrão da tela principal e cole o script abaixo. 
Atenção: Substitua os textos em letras maiúsculas nas linhas 6, 7 e 8 pelos seus IDs reais.

```javascript
function gerenciarUpload() {
  var scriptProperties = PropertiesService.getUserProperties();
  var tempoInicio = Date.now();
  var limiteTempo = 4 * 60 * 1000; // O script vai parar aos 4 minutos

  var idPastaOrigem = 'COLE_AQUI_O_ID_DA_MEET_RECORDINGS';
  var idPastaDestino = 'COLE_AQUI_O_ID_DA_PASTA_DE_PROCESSADOS';
  var idPlaylist = 'COLE_AQUI_O_ID_DA_PLAYLIST';

  var uploadUrl = scriptProperties.getProperty('uploadUrl');
  var idArquivo = scriptProperties.getProperty('idArquivo');
  var inicioByte = parseInt(scriptProperties.getProperty('inicioByte')) || 0;

  var pastaOrigem = DriveApp.getFolderById(idPastaOrigem);
  var pastaDestino = DriveApp.getFolderById(idPastaDestino);
  var arquivo;

  if (!uploadUrl) {
    var arquivos = pastaOrigem.getFilesByType('video/mp4');
    if (!arquivos.hasNext()) {
      Logger.log('Nenhum vídeo novo encontrado na pasta.');
      return;
    }
    
    arquivo = arquivos.next();
    idArquivo = arquivo.getId();

    var nomeOriginal = arquivo.getName();
    var partes = nomeOriginal.split(' - ');
    var tituloLimpo = nomeOriginal;

    if (partes.length >= 2) {
      var dataCriacao = arquivo.getDateCreated();
      var dia = ("0" + dataCriacao.getDate()).slice(-2);
      var mes = ("0" + (dataCriacao.getMonth() + 1)).slice(-2);
      var ano = dataCriacao.getFullYear();
      tituloLimpo = partes[0] + ' - ' + partes[1] + ' - ' + dia + '-' + mes + '-' + ano;
    }

    var token = ScriptApp.getOAuthToken();
    var metadata = {
      snippet: { title: tituloLimpo, categoryId: '22' },
      status: { privacyStatus: 'unlisted' }
    };

    var urlInicio = "[https://www.googleapis.com/upload/youtube/v3/videos?uploadType=resumable&part=snippet,status](https://www.googleapis.com/upload/youtube/v3/videos?uploadType=resumable&part=snippet,status)";
    var opcoesInicio = {
      method: "post",
      contentType: "application/json",
      headers: { "Authorization": "Bearer " + token },
      payload: JSON.stringify(metadata),
      muteHttpExceptions: true
    };

    var respostaInicio = UrlFetchApp.fetch(urlInicio, opcoesInicio);
    var headersResposta = respostaInicio.getHeaders();
    uploadUrl = headersResposta['Location'] || headersResposta['location'];

    if (!uploadUrl) {
      Logger.log('Erro ao criar túnel com o YouTube.');
      return;
    }

    scriptProperties.setProperty('uploadUrl', uploadUrl);
    scriptProperties.setProperty('idArquivo', idArquivo);
    scriptProperties.setProperty('inicioByte', '0');
    inicioByte = 0;
  } else {
    arquivo = DriveApp.getFileById(idArquivo);
  }

  var tamanhoTotal = arquivo.getSize();
  var token = ScriptApp.getOAuthToken();
  var tamanhoPedaco = 5 * 1024 * 1024; 

  while (inicioByte < tamanhoTotal) {
    if (Date.now() - tempoInicio > limiteTempo) {
      Logger.log('Pausa estratégica. Paramos no byte: ' + inicioByte);
      scriptProperties.setProperty('inicioByte', inicioByte.toString());
      criarGatilhoContinuacao();
      return;
    }

    var fimByte = inicioByte + tamanhoPedaco - 1;
    if (fimByte >= tamanhoTotal) {
      fimByte = tamanhoTotal - 1;
    }

    var urlDrive = "[https://www.googleapis.com/drive/v3/files/](https://www.googleapis.com/drive/v3/files/)" + idArquivo + "?alt=media";
    var opcoesDrive = {
      headers: {
        "Authorization": "Bearer " + token,
        "Range": "bytes=" + inicioByte + "-" + fimByte
      },
      muteHttpExceptions: true
    };
    var respostaDrive = UrlFetchApp.fetch(urlDrive, opcoesDrive);
    var pedaco = respostaDrive.getBlob().getBytes();

    var opcoesUpload = {
      method: "put",
      headers: {
        "Content-Range": "bytes " + inicioByte + "-" + fimByte + "/" + tamanhoTotal
      },
      payload: pedaco,
      muteHttpExceptions: true
    };

    var respostaUpload = UrlFetchApp.fetch(uploadUrl, opcoesUpload);
    var codigoResposta = respostaUpload.getResponseCode();

    if (codigoResposta === 308) {
      inicioByte = fimByte + 1;
    } else if (codigoResposta === 200 || codigoResposta === 201) {
      var dadosVideo = JSON.parse(respostaUpload.getContentText());
      var idVideo = dadosVideo.id;

      YouTube.PlaylistItems.insert({
        snippet: {
          playlistId: idPlaylist,
          resourceId: { kind: 'youtube#video', videoId: idVideo }
        }
      }, 'snippet');

      arquivo.moveTo(pastaDestino);
      limparRastros();
      Logger.log('Sucesso! Vídeo enviado por completo.');
      return;
    } else {
      Logger.log('Erro inesperado. Código: ' + codigoResposta);
      limparRastros();
      return;
    }
  }
}

function criarGatilhoContinuacao() {
  limparGatilhos();
  ScriptApp.newTrigger('gerenciarUpload')
    .timeBased()
    .after(1 * 60 * 1000) 
    .create();
}

function limparGatilhos() {
  var gatilhos = ScriptApp.getProjectTriggers();
  for (var i = 0; i < gatilhos.length; i++) {
    if (gatilhos[i].getHandlerFunction() === 'gerenciarUpload') {
      ScriptApp.deleteTrigger(gatilhos[i]);
    }
  }
}

function limparRastros() {
  PropertiesService.getUserProperties().deleteAllProperties();
  limparGatilhos();
}
