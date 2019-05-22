# Criando um servidor REST API

Tutorial de como criar um simples servidor de Back-end utilizando Node.js, Express e MongoDB

## Configurando o ambiente

1. Utilizar o comando a seguir no terminal
    ```bash
    npm init -y
	```
2. Instalar as dependências **express** e **mongoose**
	```bash
	npm i express mongoose
	```
3. Instalar as dependências de desenvolvimento **dotenv** e **nodemon**
    ```bash
    npm i --save-dev dotenv nodemon
	```
4. No arquivo *package.json* atualizar o script para:
	```json
	"scripts": {
		"devStart": "nodemon server.js"
	}
	```

5. Criar os arquivos *.env* e *.gitignore*. No arquivo *.gitignore* acrescentar as seguintes linhas:
	```
	.env
	node_modules
	```

## Criando o servidor

### > Iniciando um servidor

Para iniciar um servidor simples com express, basta adicionar as seguintes linhas:
```javascript
const express = require('express');
const app = express();

app.listen(3000, () => {
	console.log('Server Started')       
})
```

No terminal, iniciar atraves do script que criamos acima:
```bash
npm run devStart
```

O servidor estará rodando em http://localhost:3000/. Graças ao nodemon o servidor ficará atualizando toda vez que fizermos alguma alteração nos arquivos.

### > Configurando o banco de dados MongoDB através do Mongoose

Primeiramente devemos requerir a biblioteca mongoose, que nos ajudará a conectar com o MongoDB. Utilizamos essa biblioteca para nos conectarmos ao banco de dados com o comando **.connect()** e passar como parametros o link de conexão (**mongodb://localhost/NOME_DO_BANCO**) e **{ useNewUrlParser: true }**.

```javascript
const mongoose = require('mongoose');

mongose.connect('mongodb://localhost/NOME_DO_BANCO', 
		{ useNewUrlParser: true })
``` 

Para verificar se o banco esta conectado ou há erros na conexão:

```javascript	
const db = mongoose.connection
db.on('error', (error) => console.error(error))
db.once('open', () => console.log('Connected to Database'))
```

Como não é interessante deixar o link de conexão do banco de dados dentro de server.js, no arquivo *.env* criaremos uma variavel com o link para o db:

	DATABASE_URL = mongodb://localhost/NOME_DO_BANCO

A seguir no arquivo *server.js* daremos um require em **dotenv** e trocaremos o link pela variavel que criamos anteriormente:

```javascript
require('dotenv').config()

mongoose.connect(process.env.DATABASE_URL, { useNewUrlParser: true })
```

Agora que estamos com o servidor rodando e o banco de dados conectado, começaremos a criar as rotas e models e configuraremos o servidor para aceitar json.

### > Habilitando JSON (JavaScript Object Notation)

O comando **app.use()** nos permite usar qualquer tipo de middleware em nosso servidor. Middleware é o código que executa quando o servidor recebe uma requisição mas antes de passar para as rotas.

```javascript
app.use(express.json());
```
### <a id="model"></a> > Criando o Model

1. Crie uma pasta chamada *models*.
2. Crie um arquivo para o model, neste caso *subscriber.js*
3. No arquivo, requerir o uso do mongoose:
	```javascript
	const mongoose = require('mongoose')
	```
4. Crie um schema:
	```javascript
	const subscriberSchema = new mongoose.Schema({
		name: {
			type: String,
			required: true
		},
		subscribedToChannel: {
			type: String,
			required: true
		},
		subscribeDate: {
			type: Date,
			required: true
			default: Date.now				
		}
	})
	```

5. Exportar o módulo no final do arquivo:

	```javascript
	module.exports = mongoose.model('Subscriber', subscriberSchema)
	```

### > Setando as rotas

No nosso arquivo principal *server.js* chamaremos o arquivo de rotas, neste exemplo, o *routes/subscribers*.

```javascript
const subscribersRouter = require('./routes/subscribers')
```

Criaremos, na nossa pasta raiz, uma pasta chamada *routes* e dentro dessa pasta, o arquivo *subscribers.js*.

Após criarmos os arquivos, definiremos a rota através do **app.use()**.

```javascript
app.use('/subscribers', subscribersRouter)
```
### <a id="middleware"></a> > Criando um Middleware para buscar um Registro por ID

Como utilizaremos o ID para fazer varias requisições, criaremos uma função middleware para realizar essa busca do registro.

No arquivo *routes/subscribers.js* criaremos a seguinte função.

```javascript
async function getSubscriber(req, res, next) {
	let subscriber
	try{
		subscriber = await Subscriber.findById(req.params.id)
		if (subscriber == null){
			return res.status(404).json({ message: "Cannot find subscriber"})
		}
	}catch(err){
		return res.status(500).json({ message: err.message })
	}

	res.subscriber = subscriber
	next()
}
```
Nota: **Subscriber** é o [model](#model) que criamos acima, para adicioná-lo à nossa rota verifique [abaixo](#carregamentomodel)
### > Configurando as rotas

Para fazer a configuração inicial do *subscriber.js*:

```javascript
const express = require('express')
const router = express.Router()

module.exports = router
```

<a id="carregamentomodel"></a> Carregaremos o Model que criamos acima:

```javascript
const Subscriber = require('../models/subscriber')
``` 

Precisaremos configurar as seguintes rotas:

- [Getting All (Buscar Todos)](#gettingall)
- [Getting One (Buscar Um)](#gettingone)
- [Creating One (Criar)](#create)
- [Updating One (Atualizar)](#update)
- [Deleting One (Apagar)](#delete) 

#### <a id="gettingall"></a> Getting All - Buscar Todos
Na busca geral, utilizaremos um **GET** e não passaremos nenhum parâmetro.

```javascript
router.get('/', async (req, res) => {
	try {
		const subscribers = await Subscriber.find()
		res.json(subscribers);
	} catch (err) {
		res.status(500).json({ message: err.message })
	}
})
```

Como podemos verificar, foi utilizada uma função assíncrona, pois estaremos nos comunicando com o banco de dados..

#### <a id="gettingone"></a> Getting One - Buscar Um
Na busca unitária, utilizaremos um **GET** e passaremos o ID do registro que estamos buscando. Para buscar o registro, utilizaremos a função [middleware](#middleware) que criamos acima.

```javascript
router.get('/:id', getSubscriber, (req, res) => {
    res.json(res.subscriber)
})
```

#### <a id="create"></a> Creating One - Criar
Na criação de um novo registro, utilizaremos o método **POST** e não passaremos nenhum parâmetro

```javascript
router.post('/', async (req, res) => {
	const subscriber = new Subscriber({
		name: req.body.name,
		subscribedToChannel: req.body.subscribedToChannel
	})

	try {
		const newSubscriber = await subscriber.save()
		res.status(201).json(newSubscriber)
	} catch (err) {
		res.status(400).json({ message: error.message })
	}
})
```

#### <a id="update"></a> Updating One - Atualizar
Na atualização de um registro, utilizaremos o **PATCH**, pois iremos atualizar apenas uma parte do registro e não o registro inteiro (PUT). Passaremos o ID do registro que gostariamos de atualizar como parâmetro. Para a atualização também utilizaremos a função [middleware](#middleware) que criamos acima.

```javascript
router.patch('/:id', getSubscriber, async (req, res) => {
    if (req.body.name != null) {
        res.subscriber.name = req.body.name
    }
    if (req.body.subscribedToChannel != null) {
        res.subscriber.subscribedToChannel = req.body.subscribedToChannel
    }
    try {
        const updatedSubscriber = await res.subscriber.save()
        res.json(updatedSubscriber)
    } catch (err) {
        res.status(400).json({ message: err.message })
    }
})
```

#### <a id="delete"></a> Deleting One - Apagar
Para apagar um registro, passaremos o ID do registro como parâmetro e usaremos o método **DELETE**. Para a exclusão também utilizaremos a função [middleware](#middleware) que criamos acima.

```javascript
router.delete('/:id', getSubscriber, async (req, res) => {
    try {
        await res.subscriber.remove()
        res.json({ message: "Subscriber deleted successfully" })
    } catch (err) {
        res.status(500).json({ message: err.message })
    }
})
```

## Fazendo requisições REST pelo Visual Studio Code

Primeiramente faça o Download da extensão **REST Client** no Visual Studio Code.

Para fazer requisições através do VSC, crie um arquivo com a extensão *.rest* ou *.http*.

Dentro desse arquivo escreva as requisições conforme seja necessário. Segue exemplo:

	GET http://localhost:3000/subscribers

 Para mais informações vide [documentação](https://marketplace.visualstudio.com/items?itemName=humao.rest-client).

## Códigos de Erro

- 200 - Sucesso

- 201 - Objeto criado com sucesso 
- 400 - Houve um erro com a requisição do usuário. i.e Enviou dados de forma incorreta.
- 500 - Significa que houve um erro no servidor

-----

fonte: [Web Dev Simplified](https://www.youtube.com/watch?v=fgTGADljAeg "video")