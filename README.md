## Persistência de dados com SQLite
Hoje iremos falar sobre SQLite, e realizar uma atividade prática para reforçarmos nosso entendimento sobre Flutter e neste momento, o SQLite.
Após as considerações iniciais, vocês com base neste tutorial, irão criar a estrutura necessária para poder utilizar persistência de dados local em Flutter.


### Widget Main
Como de praxe, criem um novo projeto e configurem o Widget Main.

```dart
import 'package:flutter/material.dart';

void main() {
  runApp(const MyApp());
}

class MyApp extends StatelessWidget {
  const MyApp({super.key});

  @override
  Widget build(BuildContext context) {
    return MaterialApp(
      theme: ThemeData(primarySwatch: Colors.green),
      home: const HomePage(),
    );
  }
}

```

### Página Principal
Crie o arquivo `lib\pages\home\home_page.dart`.

```dart
import 'package:flutter/material.dart';

class HomePage extends StatefulWidget {
  @override
  _HomePageState createState() => _HomePageState();
}

class _HomePageState extends State<HomePage> {

  @override
  Widget build(BuildContext context) {
    return Scaffold(
      appBar: AppBar(
        title: Text('SQLite Contacts'),
      ),
      body: Center(child: Text('Body')),
      floatingActionButton: FloatingActionButton(
        child: Icon(Icons.add),
        onPressed: () {},
      ),
    );
  }
```

# SQLite
Como nós já sabemos, o SQLite é um banco de dados simples, robusto baseado em arquivo e não necessita de um SGBD; por estes e outras qualidades, é o banco de dados mais utilizados em mobile e sistemas embarcados.

## SQFLite
O pacote `sqflite` no possibilida a mesma sintaxe de gerenciamento de banco de dados que já vimos no Android com o `OpenSQLiteHelper`, mas em Flutter, é um pouco mais simples.


### Adicione o SQFlite ao projeto

> Atenção neste passo. Se estiver com SDK atualizado, adicione a versão ao pubespec, senão inclua a dependencia via linha de comando.

1. Abra o arquivo `pubspec.yaml` e adicione o trecho abaixo:
```yaml
  sqflite: ^2.2.0+3
```

ou digite o comando abaixo no shell

```shell
flutter pub add sqflite
```

### Crie o SQLiteOpenHelper
Criaremos a classe `SQLiteOpenHelper`, esta classe irá facilitar o gerenciamento do código necessário para criarmo um App simples que utilize persistência de dados em SQLite.

1. Crie a classe `lib\helper\sqlite_helper.dart` no diretório especificado.
2. Adicione as dependências ao arquivo `sqlite_helper.dart`.

```dart
import 'package:sqflite/sqflite.dart';
import 'package:path/path.dart';
import 'dart:async';
```
3. Crie as constantes para simplificar nossa codificação um pouco mais adiante.

```dart
// Constantes para facilitar a criacao de tabela e manipular atributos
final String tabela = 'contatos';
final String colunaId = 'id';
final String colunaNome = 'nome';
final String colunaEmail = 'email';
final String colunaTelefone = 'telefone';
final String colunaCaminhoImagem = 'caminhoImagem';
```

#### Model
Vamos criar a model class para trabalharmos com os dados de nossos contatos. O Model irá simplificar o acesso aos dados no SQLite.

1. Crie a Model.

```dart
// Model Class
class Contato {
  int? id;
  String? nome;
  String? email;
  String? telefone;
  String? caminhoImagem;
}
```

2. Adicione o contrutor.

```dart
// Model Class
class Contato {
  int? id;
  String? nome;
  String? email;
  String? telefone;
  String? caminhoImagem;

  // Construtor com parametros nomeados
  Contato({this.nome, this.email, this.telefone, this.caminhoImagem});
}

```

3. Adicione os métodos conversores:

```dart
// Model Class
class Contato {
  int? id;
  String? nome;
  String? email;
  String? telefone;
  String? caminhoImagem;

  // Construtor com parametros nomeados
  Contato({this.nome, this.email, this.telefone, this.caminhoImagem});

  //Metodo converte objeto Contao em Mapa
  Map<String, dynamic> toMap() {
    Map<String, dynamic> map = {
      colunaNome: nome,
      colunaEmail: email,
      colunaTelefone: telefone,
      colunaCaminhoImagem: caminhoImagem
    };

    if (id != null) {
      map[colunaId] = id;
    }
    return map;
  }

  //Metodo que converte Mapa em objeto Contato
  Contato.fromMap(Map<String, dynamic> map) {
    id = map[colunaId];
    nome = map[colunaNome];
    email = map[colunaEmail];
    telefone = map[colunaTelefone];
    caminhoImagem = map[caminhoImagem];
  }
}
```
#### Acesso ao Banco SQLite
Precisamos com base no framework SQFLite, criar os métodos que possam gerenciar o banco de dados, ou seja, criar e manipular bancos, tabelas e registros. Para tal, vamos criar uma função que inicializará o banco e criará a tabela contatos, semelhante ao que já fizemos em Android.

1. Dentro do mesmo arquivo, crie a classe `SQLiteOpenHelper` e adicione o construtor com o Pattern Factory e adicione uma variavel do tipo Database.

```dart
//Como em Android, criaremos uma classe Helper para facilitar o uso do SQLite pelo
class SQLiteOpenHelper {
  //Singleton para SQLiteOpenHelper
  static final SQLiteOpenHelper _instance = SQLiteOpenHelper._internal_();
  factory SQLiteOpenHelper() => _instance;

  SQLiteOpenHelper._internal_();

  Database? _dataBase;
}
```

2. Adicione o trecho abaixo da variável _dataBase:

```dart
//Método responsavel por inicializar o banco de dados criar as tabelas necessárias
Future<Database> inicializarBanco() async {
  // 1
  final databasePath = await getDatabasesPath();
  //2
  final path = join(databasePath, "contatos.db");

//3
  return await openDatabase(path, version: 1,
      onCreate: (Database db, int version) {
    db.execute('''
          CREATE TABLE IF NOT EXISTS $tabela(
            $colunaId INTEGER PRIMARY KEY,
            $colunaNome TEXT NOT NULL,
            $colunaEmail TEXT NOT NULL,
            $colunaTelefone TEXT,
            $colunaCaminhoImagem TEXT
          );
        ''');
  });
}
```
O trecho acima respectivamente:

> //1 - Obtem o caminho onde o banco será armazenado no dispositivo;  

> //2 - Contaceta o nome do banco ao caminho obtido.  

> //3 - Realiza chamada ao `openDatabase` que com base no path, cria o arquivo do banco de dados e executa a instrução de criação da tabela de forma assíncrona, notem as palavras reservadas [async e await].  

3. Adicione o getter que garantirá uma instancia única da referencia de nosso banco de dados.

```dart
//Getter para instancia única de referencia para Banco de Dados sqlite
Future<Database> get dataBase async {
  if (_dataBase != null) {
    return _dataBase;
  } else {
    return _dataBase = await inicializarBanco();
  }
}

```

## CRUD
Com os métodos de acesso ao banco e o nosso model e seus conversores criados, vamos criar as funcionalidades que irão manipular o banco de dados em si.

#### CREATE
Inclua o método de inserção de novo registro na entidade.

```dart
//Método de insercao de registro na tabela contato
Future<Contato> insert(Contato contato) async {
  //1
  Database? db = await dataBase;
  //2
  contato.id = await db?.insert(tabela, contato.toMap());
  return contato;
}

```

> //1 - Recuperamos uma referencia do banco de dados de forma assincrona.  

> //2 - Ao utilizar a referencia do banco de dados, podemos utilizar os métodos do helper; neste caso utilizamos o insert que necessita do identificador da tabela e de uma Map; com estes dados a função insere os registros no banco e retorna um objeto do tipo do nosso Model.

#### READ
Inclua o método de consulta a um registro existente na entidade.

```dart
//Método para recuperar um registro na tabela contato
Future<Contato> findById(int id) async {
  Database? db = await dataBase;
  //1
  List<Map<String, dynamic>> map = await db!.query(tabela,
      distinct: true,
      //2
      columns: [
        colunaId,
        colunaNome,
        colunaEmail,
        colunaTelefone,
        colunaCaminhoImagem
      ],
      //3
      where: '$colunaId = ?',
      //4
      whereArgs: [id]);

  return map.length > 0 ? Contato.fromMap(map.first) : Map();
}

```
> //1 - Retorno de um Map com os dados da entidade  

> //2 - Colunas do conjunto resultado  
> //3 - Clausula where informando a coluna utilizada  

> //4 - Argumentos utilizados na clausula where  

### UPDATE
Inclua o método de atualização de registros

```dart
//Método de atualizacao de registro
Future<int> update(Contato contato) async {
  Database? db = await dataBase;
  return await db.update(tabela, contato.toMap(),
      where: '$colunaId = ?', whereArgs: [contato.id]);
}

```

### DELETE
Inclua o método de remoção de registro

```dart
//Método de remocao de registro
Future<int> delete(int id) async {
  Database db = await dataBase;
  return await db.delete(tabela, where: 'id = ?', whereArgs: [id]);
}

```

### DAO - PLUS
Inclua o método para buscar todos os registros da entidade encapsulados em uma lista.

```dart
//Método respnsável por buscar todos os contatos na tabela contato
Future<List<Contato>> findAll() async {
  Database? db = await dataBase;

  List<Map> mapContatos = await db!.rawQuery('SELECT * FROM $tabela;');

  List<Contato> contatos = List();

  mapContatos.forEach((element) {
    contatos.add(Contato.fromMap(element));
  });
  return contatos;
}

```

Inclua o método para recuperar o total de registros na entidade.

```dart
// Metodo para recuperar o total de registros
Future<int> getCount() async {
  Database db = dataBase as Database;
  return Sqflite.firstIntValue(
      await db.rawQuery('SELECT COUNT(*) FROM $tabela'));
}

```

Inclua o método para fechar a "conexão com o SQLite"

```dart
Future close() {
  Database db = dataBase as Database;
  return db.close();
}
```

# That's All Folks

# Brincadeira... Temos atividade

### Formulário de Cadastro de Contato

Vocês se lembram deste formulário?

<img src="images/formulario.jpg" height="480">

A atividade de hoje será vocês criarem o `FloatingActionButton` que fará a transição para a tela acima.
Com base nesta tela, iremos uilizar a nossa classe `OpenSQLiteHelper` e persistir os dados no banco e preencher a lista de contatos na página inicial.


## AtividadeOpenHelper

### Adicionar navegação ao FloatingActionButton da tela principal [home_page.dart]
Adicione ao floatingActionButton o trecho de navegação:

```dart

import 'package:flutter/material.dart';

class HomePage extends StatefulWidget {
  @override
  _HomePageState createState() => _HomePageState();
}

class _HomePageState extends State<HomePage> {

  @override
  Widget build(BuildContext context) {
    return Scaffold(
      appBar: AppBar(
        title: Text('SQLite Contacts'),
      ),
      body: Center(child: Text('Body')),
      floatingActionButton: FloatingActionButton(
        child: Icon(Icons.add),
        onPressed: () {
          //Navegação
          MaterialPageRoute(builder: (context) => AdicionarContatoScreen()));
        },
      ),
    );
  }

```

### Criar o formulário de adição de contato.
1. Crie o arquivo `lib\screens\adicionar_contato.dart`.
2. Crie o Widget customizado de caixa de texto com ícones para utilizar no Formulário; crie o arquivo `lib\widgets\custom_textfield.dart`.
3. Adicione o trecho abaixo em `custom_textfield.dart`.

```dart
import 'package:flutter/material.dart';

class CustomTextField extends StatelessWidget {
  final String hint;
  final IconData icon;
  final TextEditingController controller;

  const CustomTextField({this.hint, this.icon, this.controller});

  @override
  Widget build(BuildContext context) {
    return Padding(
      padding: const EdgeInsets.only(left: 16, right: 16, bottom: 8),
      child: TextField(
        controller: controller,
        decoration: InputDecoration(
          prefixIcon: Icon(icon),
          labelText: hint,
          focusedBorder: UnderlineInputBorder(
            borderSide: BorderSide(color: Colors.teal.shade200),
            borderRadius: BorderRadius.all(Radius.elliptical(5, 10)),
          ),
        ),
      ),
    );
  }
}
```
4. Adicione o código do formulário ao arquivo `adicionar_contato.dart`.

```dart
class AdicionarContatoScreen extends StatefulWidget {

  @override
  _AdicionarContatoScreenState createState() => _AdicionarContatoScreenState();
}

class _AdicionarContatoScreenState extends State<AdicionarContatoScreen> {

  TextEditingController nomeController = TextEditingController();
  TextEditingController emailController = TextEditingController();
  TextEditingController telefoneController = TextEditingController();


  @override
  Widget build(BuildContext context) {
    return Scaffold(
      appBar: AppBar(
        title: Text('Adicionat Contato'),
      ),
      body: SingleChildScrollView(
        child: Column(
        crossAxisAlignment: CrossAxisAlignment.stretch,
        children: [
          CustomTextField(hint: "Nome", icon: Icons.text_fields, controller: nomeController,),
          CustomTextField(hint: "E-mail", icon: Icons.email, controller: emailController,),
          CustomTextField(hint: "Telefone", icon: Icons.phone, controller: telefoneController,),
          Padding(
            padding: const EdgeInsets.all(16.0),
            child: RaisedButton(
              shape: RoundedRectangleBorder(
                borderRadius: BorderRadius.circular(20),
              ),
              color: Colors.teal.shade200,
              onPressed: () {},
              child: Text('Salvar', style: TextStyle(color: Colors.black)),
            ),
          ),
        ],
      ),
      ),
    );
  }
}

```


# Adicionar o SQLite
Vamos adicionar as funcionalidade dos SQLite incluindo o código necessário para a persistência de dados.

1. Altere o construtor `AdicionarContatoScreen` adicionando na linha 7, o código abaixo.

```dart
final Contato contato;
AdicionarContatoScreen({this.contato});
```
A partir de agora, ele pode receber um objeto contato em sua criação, será útil para a edição.

## Adicionar funcionalidade ao botão Salvar

Dentro do callback do RaisedButton Salvar, na propriedade onPressed, adicione apenas o conteúdo entre as chaves:

```dart
onPressed: () {
  var contato = Contato(
      nome: nomeController.text,
      email: emailController.text,
      telefone: telefoneController.text);
  if (contatoEdicao.nome != null) contato.id = contatoEdicao.id;
  Navigator.pop(context, contato);
},
```

Neste trecho estamos simplesmente preenchendo o objeto contato e fazendo sua devolução para a tela anterior; a main_sceeen irá fazer a persistencia dos dados, até conhecermos outros Frameworks para trabalhar com fluxo de dados entre telas nas proximas aulas; neste momento `Navigator.pop(context, contato);` está devolvendo um objeto contato preenchido para a `HomePage()`.

## Agora sim, adicionar o SQLite!

Precisamos ajustar o recebimento do objeto contato para podermos armazená-lo, então primeiramente vamos refatorar a chamada da tela de cadastro.

1. Acesse a classe `home_page.dart`.
2. Crie uma instância do nosso Helper.
2. Adicione ao final da classe o método obterTodosContatos(), lembre-se no final da classe, mas dentro da classe.

```dart
class HomePage extends StatefulWidget {
  @override
  _HomePageState createState() => _HomePageState();
}

class _HomePageState extends State<HomePage> {
  //Adicione neste ponto do código o SQLiteOpenHelper
  final SQLiteOpenHelper helper = SQLiteOpenHelper();
  //Crie uma lista para armazenar os dados vindos do banco de dados.
  List<Contato> listContatos = List();
//...
}

```

3. Crie a função que irá buscar todos registros no banco de dados.

```dart

obterTodosContatos() {
  helper
      .findAll()
      .then((list) => setState(() {
            listContatos = list;
          }))
      .catchError((error) {
    print('Erroa ao recuperar contatos: ${error.toString()}');
  });
}

```

4. Inclua o método obterTodosContatos ao initState() para garantir que a página será sempre atualizada.

```dart
class _HomePageState extends State<HomePage> {
  //Adicione neste ponto do código o SQLiteOpenHelper
  final SQLiteOpenHelper helper = SQLiteOpenHelper();
  //Crie uma lista para armazenar os dados vindos do banco de dados.
  List<Contato> listContatos = List();
  @override
  void initState() {
    super.initState();
    obterTodosContatos();
  }
//...
}
```



Utilizando o Helper, é executada a função findAll() que trás todos os registros em uma List<Contato> e o setState() irá atualizar a interface gráfica.

5. Atualize o código que irá fazer a transição de telas e persistencia de dados. Acima da função adicione o trecho abaixo:

```dart

_exibirPaginaContato({Contato contato}) async {
  final retornoContato = await Navigator.push(context,
      MaterialPageRoute(builder: (context) => AdicionarContatoScreen(contato: contato)));

  if (retornoContato != null) {
    if (contato != null) {
      await helper.update(retornoContato);
    } else {
      await helper.insert(retornoContato);
    }
    obterTodosContatos();
  }
}

```  

6. Ajuste o callback do FloatingActionButton no arquivo `home_page.dart`. Este método irá receber o objeto Contato da tela de inclusão e irá persistí-lo. 

```dart
    floatingActionButton: FloatingActionButton(
        child: Icon(Icons.add),
        onPressed: () {
          //Navegação
          _exibirPaginaContato();
        },
      ),
    );
```


Este trecho ao fazer a transição, envia um novo contato vazio pelo construtor e ao receber um referencia já preenchida no formulário, armazena em `retornoContato` e valida se o contato é nulo.
Se o contato não for nulo e for diferente de vazio um contato será criado, caso contrário o contato será atualizado, tudo isto de forma simplificada pelo Helper.
Logo abaixo, uma lista de contatos é preenchida e uma lista será atualizada em nosso UI.
Vamos criar esta lista agora.

## Criar lista de contatos para exibição.

1. Acesse `HomePage` e substitua o código do body por este:

```dart
body: ListView.builder(
  itemBuilder: (context, index) {
    return GestureDetector(
      child: Card(
        elevation: 4,
        child: Padding(
          padding: EdgeInsets.all(16),
          child: Row(
            children: [
              Image.asset(
                'assets/images/social.png',
                width: 80,
                height: 80,
              ),
              Padding(
                padding: const EdgeInsets.only(left: 16.0),
                child: Column(
                  crossAxisAlignment: CrossAxisAlignment.start,
                  children: [
                    Text(
                      listContatos[index].nome,
                      style: TextStyle(
                          fontWeight: FontWeight.bold, fontSize: 24),
                    ),
                    Text(
                      listContatos[index].email,
                      style: TextStyle(fontSize: 18),
                    ),
                    Text(
                      listContatos[index].telefone,
                      style: TextStyle(fontSize: 18),
                    ),
                  ],
                ),
              ),
            ],
          ),
        ),
      ),
      onTap: () {
        _exibirPaginaContato(contato: listContatos[index]);
      },
    );
  },
  itemCount: listContatos.length,
),

```  
O ListView.builder, como vimos na aula anterior, é responsável por criar uma coleção de Widgets e fazer o `binding` de dados em cada um dos elementos desta coleção.
O método carrega uma lista com os dados do banco e a partir dela, preenchemos os objetos da lista pelo callback `itemBuilder:`.
Estamos utilizando uma imagem comum a cada contato; ela está no diretório `images` aqui dentro do gitlab.
O próximo passo será adicionar as fotografias aos contatos.


# Adicionar Fotografias ao Contatos.

Vamos adicionar a capacidade de armazenar arquivos no dispositivo e recuperar uma referencia a estes arquivos utilizando o SQLite.
Iremos armazenar fotografias com o pacote `Image Picker` do `pub.dev`, este pacote consegue de forma simplificada recuperar o caminho para os arquivos armazenados nos projetos Flutter; assim poderemos utilizar estas funcionalidades no cadastro de serviços no Projeto Interdisciplinar também.

## Adicionar o pacotes `image_picker` ao projeto.

1. Abra a arquivo `pubspec.yaml` e adicione logo o trecho do image picker, com referencia ao trecho abaixo:

> Lembre-se de utilizar a melhor versão para o seu ambiente com a inclusao via linha de comando.

```yaml
  cupertino_icons: ^1.0.2
  sqflite: ^2.2.0+3
  image_picker: ^0.8.6
```

2. Salve o arquivo ou vá ao terminal e execute o comando `pub get`.

## Atualizar a tela de adição de contatos para o input de uma imagem capturada a partir da camera do dispositivo.

1. Abra o arquivo `adicionar_contato.dart`.
2. Remova ou comente a linha de código `crossAxisAlignment: CrossAxisAlignment.stretch,`.
3. Adicione o código abaixo para adicionar uma imagem circular ao topo da coluna utilizada.

```dart
SizedBox(height: 20,),
GestureDetector(
  child: Container(
    width: 150,
    height: 150,
    decoration: BoxDecoration(
      shape: BoxShape.circle,
      image: DecorationImage(
        image: contatoEdicao.caminhoImagem != null
            ? FileImage(File(contatoEdicao.caminhoImagem))
            : AssetImage('assets/images/social.png') as ImageProvider,
        fit: BoxFit.cover,
      ),
    ),
  ),
  onTap: () {

  },
),
```

4. Crie os objetos para armazenar o caminhos dos arquivos das imagens a serem armazenadas; adicione o trecho abaixo logo abaixo dos `TextEditingControllers`

```dart
Contato contatoEdicao;
final ImagePicker imagePicker = ImagePicker();
PickedFile _caminhoImagem;
```

5. Adicione ao callback `onTap(){}` os comandos necessários para podermos fazer a captura de imagem da camera.

```dart

onTap: () {
  /* De forma assincrona, o objeto imagePicker gerencia a abertura do aplicativo de camera selecionado,
    persiste a forografia e retorna o caminho do arquivo armazenado.
  */
  imagePicker.getImage(source: ImageSource.camera).then((value) {
    // Atualizamos os dados da tela.
    setState(() {
      _caminhoImagem = value;
      //Depois de armazene globalmente a referencia do arquivo, atualizamos o atributo caminhoImagem do contato, para recupera o arquivo para exibiçao.
      contatoEdicao.caminhoImagem = _caminhoImagem.path;
    });
  });
},

```
6. Atualize o método que salva o contato adicionando a referencia para o arquivo da fotografia. No callbak no RaisedButton Salvar, adicione:

```dart
onPressed: () {
  Contato contato = Contato(
      nome: nomeController.text,
      email: emailController.text,
      telefone: telefoneController.text,
      caminhoImagem: _caminhoImagem.path);
  if (contatoEdicao.nome != null) {
    contato.id = contatoEdicao.id;
  }
  Navigator.pop(context, contato);
},

```

Notem que adicionamos apenas o caminho para o arquivo da imagem.

## Ajustar a exibição dos Cards para exibir as fotografias.
Agora que nossos contatos podem conter fotografias, precisamos ajustar sua exibição de forma que os contatos que possuam fotografias possam exibi-las.

1. Abra o arquivo `home_page.dart`.
2. Dentro da propriedade `itemBuilder`, substitua o widget Image.asset() por um Container personalizado para a exibição da imagem do contato. Adicione o código abaixo em substituição ao bloco do Image.asset().

```dart
Container(
  width: 80,
  height: 80,
  decoration: BoxDecoration(
    shape: BoxShape.circle,
    image: DecorationImage(
      image: listContatos[index].caminhoImagem != null
          ? FileImage(
              File(listContatos[index].caminhoImagem))
          : AssetImage('assets/images/social.png') as ImageProvider,
      fit: BoxFit.cover,
    ),
  ),
),
```

## Preencher os dados dos contatos ao realizar a transição entre a tela de exibição e o formulário.
Para que possamos visualizar e editar os dados dos contatos, vamos carregá-los ao navegar entre as telas.
Vamos alterar a tela de adição de contatos para preencher os campos de entrada de texto com os dados do contato.

1. Abra o arquivo `adicionar_contato.dart`.
2. Altere o método `initState` com o código:

```dart
@override
void initState() {
  super.initState();
  if (widget.contato == null) {
    contatoEdicao = Contato();
  } else {
    contatoEdicao = Contato.fromMap(widget.contato.toMap());
    nomeController.text = contatoEdicao.nome;
    emailController.text = contatoEdicao.email;
    telefoneController.text = contatoEdicao.telefone;
  }
}
```

Este trecho irá verificar se recebemos um contato nulo que acarretaria na criação de um novo contato ou se recebemos um contato não nulo, o que acarretaria a atualização de um contato existente.
Este trecho irá preencher o formulário com os dados do contato, vaso ele já exista.



# That´s All Folks

