<h2> index.js</h2>

// importando a biblioteca de api do telegram 
const TelegramBot = require('node-telegram-bot-api');

// importando nosso arquivo que faz a chamada para o dialogflow
const dialogflow = require('./dialogflow');

// importando nosso módulo que faz a busca no youtube
const youtube = require('./youtube');

// token recebido pelo bot father
const token = '12987069763@bootbot';

// nova instância do telegram
const bot = new TelegramBot(token, { polling: true });

// escuta mensagens enviadas pelos usuários
bot.on('message', async (msg) => {
    
    // id do chat do usuário
    const chatId = msg.chat.id;

    // resposta do dialogflow
    const dfResponse = await dialogflow.sendMessage(chatId.toString(), msg.text);

    // texto a partir da resposta do dialogflow
    let textResponse = dfResponse.text;
    
    // verifica a intenção a partir da resposta do dialogflow
    if (dfResponse.intent === 'Treino específico') {
        // modifica o texto para os dados retornados a partir da busca realizada no youtube
        // lembre-se que para acessar o campo corpo dentro de fields ele teve que ser definido como uma entidade no dialogflow
        textResponse = await youtube.searchVideoURL(textResponse, dfResponse.fields.corpo.stringValue);
    }
    
    // envio da mensagem para o usuário do telegram
    bot.sendMessage(chatId, textResponse);
});

<h2> DialogoFlow</h2>
// importando a biblioteca do dialogflow
const dialogflow = require('dialogflow');

// importando o arquivo contendo as configs do dialogflow
const configs = require('./configs/agent.json');

// criando uma sessão para a aplicação de acordo com as credenciais
const sessionClient = new dialogflow.SessionsClient({
    projectId: configs.project_id,
    credentials: { private_key: configs.private_key, client_email: configs.client_email }
});

// funcao para encapsular o envio de mensagens do telegram para o dialogflow
async function sendMessage(chatId, message) {
    // criando ou recuperando a sessão do usuário
    const sessionPath = sessionClient.sessionPath(configs.project_id, chatId);

    // objeto para montar o request para o dialogflow
    const request = {
        session: sessionPath,
        queryInput: {}
    };

    // request para tipo texto
    const textQueryInput = { text: { text: message, languageCode: 'pt-BR' } };

    // request para tipo evento
    const eventQueryInput = { event: { name: 'start', languageCode: 'pt-BR' } }

    // verificando se a mensagem enviada foi um start caso seja monta um evento chamando a action 'start'
    // lembrem-se que essa action precisa estar cadastrada no dialogflow para conseguirmos chamá-la
    request.queryInput = message === '/start' ?  eventQueryInput : textQueryInput;

    // respostas da requisição para o dialogflow
    const responses = await sessionClient.detectIntent(request);

    // resultado da resposta do dialogflow
    const result = responses[0].queryResult;

    // retornando objeto para ser utilizado no arquivo index.js
    return { text: result.fulfillmentText, intent: result.intent.displayName, fields: result.parameters.fields };
}

// exportando a função sendMessage
module.exports.sendMessage = sendMessage

<h2>youtube.js</h2>
<h1>youtube.js</h1>
 youtube.setKey(config.key);

// importando a biblioteca de api do youtube
const YouTube = require('youtube-node');

// importando as configurações para serem usadas pela biblioteca do youtube
const config = require('./configs/youtube.json');

// criando uma instância do youtube
const youtube = new YouTube();

// setando a key na instância do youtube para conseguirmos fazer as pesquisas
youtube.setKey(config.key);

// função para busca de vídeos
function searchVideoURL(message, queryText) {
  // encapsulando função de search em uma promise para conseguirmos retornar os resultados do callback
    return new Promise((resolve, _) => {
      // realizando a busca baseada na concatenação da sring e do queryText
        youtube.search(`Exercício em casa para ${queryText}`, 2, function(error, result) {
            if (error) {
              // caso ocorra algum erro essa mensagem será retornada
              // lembrem-se que a melhor prática seria rejeitar essa promise nesse ponto
              resolve('Não foi possível encontrar um vídeo no momento, por favor tente mais tarde');
            } else { 
              // gerando um array de videoIds
              const videoIds = result.items.map((item) => item.id.videoId).filter(item => item);

              // gerando um array de links do youtube
              const youtubeLinks = videoIds.map(videoId => `https://www.youtube.com/watch?v=${videoId}`).join(', ');

              // retornando a mensagem inicial concatenadas com os links do youtube para o arquivo index.js
              resolve(`${message} ${youtubeLinks}`);
            }
          });
    });
}


