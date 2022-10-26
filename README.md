# Previsão do Tempo

## Objetivo

O objetivo desta atividade é aprender programação assíncrona em Dart. Vamos trabalhar com a obtenção
da localização do dispositivo (GPS) e rede - busca de dados pela internet.

## Implementação

Para aplicarmos esses conhecimentos, vamos construir um aplicativo de previsão do tempo.

### Código base

Este repositório fornece para você os códigos-base para o desenvolvimento:

* lib
  * screens
    * city_screen.dart - código para a tela de seleção de cidade para obtenção da previsão de tempo.
    * loading_screen.dart - código para a tela de busca de localização pelo GPS.
    * location_screen.dart - código para a "tela principal", que traz a previsão do tempo para a cidade atual.
  * services
    * location.dart - conterá o código para localização via GPS.
    * networking.dart - conterá o código para busca dos dados na api de tempo.
    * weather.dart - contém constantes que serão mostradas para o usuário.
  * utilities
    * constants.dart - contém constantes para padronização de estilos e redução de escrita de código.

Tome um tempo para analisar os arquivos antes de começarmos.

## Código

### Obter localização por vários dispositivos

Começaremos por obter a informação de GPS. Para isso, utilizaremos um plugin do flutter,
chamado geolocator:
https://pub.dev/packages/geolocator

Esse plugin busca a localização, seja no android ou iOS.

Para instalá-lo, abra seu `pubspec.yaml` e procure a linha com `cupertino_icons`.
*Abaixo* dessa linha, inclua: `geolocator: ^9.0.2`.
*ATENÇÃO: mantenha a identação da linha anterior.*

Seu arquivo ficará, nessa região, assim:
```yaml
dependencies:
  flutter:
    sdk: flutter

  cupertino_icons: ^1.0.2
  geolocator: ^9.0.2
```

Após isso, execute o Pub get.

Agora vamos utilizar esse pacote no nosso projeto; por enquanto abre o arquivo
`loading_screen.dart`, impporte o geolocator:

`import 'package:geolocator/geolocator.dart';`

Então, dentro da classe `_LoadingScreenState`, crie um método `void getLocation()`, 
que conterá a linha informada pela documentação, para obtermos a localização atual do dispositivo:

```dart
void getLocation() {
  Position position = await Geolocator.getCurrentPosition(desiredAccuracy: LocationAccuracy.low);
}
```

note que ao entrar esse código, você notará que `await` está marcado como vermelho,
precisamos marcar o método getLocation como `async`.

*await* é uma palavra-chave que indica que o método em questão (getCurrentPosition) está sendo executado
assíncronamente - em segundo plano. Dessa forma, precisamos indicar ao dart que o método getLocation é assíncrono.
Métodos assíncronos servem para executarmos tarefas que possam consumir tempo, principalmente por questões de entrada e saída
de dados, buscas via rede, etc, de forma que o dispositivo não fique **travado** aguardando o fim da execução desses métodos.

Então, altere a assinatura de getLocation: `Future<void> getLocation() async {`.

Depois disso, podemos inserir uma linha para imprimir a posição, para efeito de teste. Logo abaixo
da linha `Position position ....`, adicione a linha `print(position)`.

Por fim, necessitamos pedir autorização ao dispositivo para usarmos o GPS. Para isso, precisamos seguir a 
documentação da biblioteca. Vamos precisar alterar a configuração em dois arquivos XML.
* Android:
  * Abra o arquivo no caminho `android/app/src/main/AndroidManifest.xml`.
  * Logo abaixo da linha: `<manifest xmlns:android="http://schemas.android.com/apk/res/android" package="com.example.tempo_template">`, adicione a linha:
  
  `<uses-permission android:name="android.permission.ACCESS_COARSE_LOCATION" />`
  * Aqui informamos ao android que nosso aplicativo utilizará a permissão de localização "grosseira" (ao contrário da fina, para, por exemplo, o waze)
* iOS:
  * Abra o arquivo no caminho `ios/Runner/Info.plist`
  * Logo abaixo de `<dict>` adicione as linhas:
```xml
  <key>NSLocationWhenInUseUsageDescription</key>
  <string>This app needs access to location when open.</string>
```

Após essa alteração, vamos criar uma nova função no código de `loading_screen.dart`. Essa função deve ser criada
logo acima da função `getLocation` que acabamos de criar:

```dart
  Future<void> checkLocationPermission() async {
    bool serviceEnabled;
    LocationPermission permission;

    serviceEnabled = await Geolocator.isLocationServiceEnabled();
    if (!serviceEnabled) {
      // serviço de localização desabilitado. Não será possível continuar
      return Future.error('O serviço de localização está desabilitado.');
    }
    permission = await Geolocator.checkPermission();
    if (permission == LocationPermission.denied) {
      permission = await Geolocator.requestPermission();
      if (permission == LocationPermission.denied) {
        // Sem permissão para acessar a localização
        return Future.error('Sem permissão para acesso à localização');
      }
    }
    if (permission == LocationPermission.deniedForever) {
      // permissões negadas para sempre
      return Future.error('A permissão para acesso a localização foi negada para sempre. Não é possível pedir permissão.');
    }
  }

```

Agora, *antes* da linha `Position position ...`, em `getLocation`, inclua uma chamada para essa
nova função: `await checkLocationPermission();`.

No fim, a função `getLocation` ficará assim:
```dart
Future<void> getLocation() async {
    // Verificando permissão de acesso
    await checkLocationPermission();

    // agora podemos pedir a localização atual!
    Position position = await Geolocator.getCurrentPosition(desiredAccuracy: LocationAccuracy.low);
    print(position);
  }
```

Execute seu código no emulador. Pressione o botão azul, note que o android solicitará permissão de localização.
Depois, observe a janela de console, Você verá impressa uma latitude e longitude. Esses valores
são configuráveis no emulador. 

Você pode clicar nos "três pontos" sobre a janela do emulador.


Uma nova janela aparecerá. Na aba Location, você verá um mapa. Pode 
navegar com o mouse pelo mapa e selecionar uma localização clicando duas vezes sobre o ponto.


Ali, você pode clicar em save point e armazenar o ponto para um teste futuro.


E você pode clicar em set location. A partir daí seu celular emulado estará enviando essa localização.


### Métodos para o ciclo de vida de um **Stateful Widget**

No momento estamos buscando a localização ao clicar no botão. Mas idealmente, queremos que
essa funcionalidade aconteça ao abrirmos o aplicativo. Para isso, precisamos entender um pouco sobre
o ciclo de vida dos **Widgets** (**Widget Lifecycle*).

#### Stateless Widgets

No caso de um Stateless Widget, O Ciclo de vida é simples. Ele é construído e destruído, quando não for mais
utilizado. Ele não tem estado. Toda mudança (por exemplo uma cor diferente) é implementada com uma destruição e reconstrução.

#### Stateful Widgets

Os Stateful Widgets também podem ser combinados e eles possuem um estado. Sabemos que têm um estado
que pode ser alterado através do método `setState`. Um Stateful Widget tem um tempo de vida maior, e
também tem mais métodos:

```dart
void initState() {
  // é disparado quando o Widget for criado
}

Widget build(BuildContext context) {
  // disparado quando o objeto for construído e o widget aparecerá na tela
  return null;
}

void deactivate() {
  // é disparado quando o widget foi destruído
}
```

### Carregando a localização *sem* que qualquer botão seja pressionado

Primeiro, remova todo o conteúdo *dentro* de `Scaffold`. Vamos tirar o botão de carregamento.

Agora, crie o método initState (ao digitar, um snippet de código aparecerá).
Crie esse método logo acima do método `build`:

```dart
@override
  void initState() {
    super.initState();
    getLocation();
  }
```

Experimente recarregar seu programa. Agora, sem que você pressione qualquer coisa, a posição será recebida.

### Refatorando o código

Idealmente, lógica de negócio, ou de serviços, não deve ser inserida na tela. Vamos refatorar esse código, para isolar lógica
de serviço do "desenho" de tela.

#### Desafio

Refatore o código do aplicativo de forma que a lógica de obter a localização atual seja 
manejada por um objeto `Location`.

* Crie uma classe `Location` no arquivo `lib/services/location.dart`;
* Essa classe deve ter dois atributos: `latitude` e `longitude`. Ambos do tipo `double`.
* Mova o método `checkLocationPermission` para a classe `Location`.
* A classe `Location` também deve ter um método `getCurrentLocation()`. Mova o código de `getLocation()` para o novo método `getCurrentLocation()`.
* O método `getCurrentLocation()` deve fazer com que os valores de `latitude` e `longitude` da `position` sejam atribuídos aos atributos `latitude` e `longitude` da classe `Location`.
* No arquivo `loading_screen.dart` atualize o `getLocation()` de forma que você:
  * Crie um novo objeto de `Location`
  * chame `getCurrentLocation()`
  * Imprima os valores armazenados em `latitude` e `longitude`
* Note que você não precisa criar um construtor para `Location`. Aqui você pode utilizar o ponto de interrogação `?` para indicar que os atributos `latitude` e `longitude` são anuláveis (só serão preenchidos após chamar-se `getCurrentLocation()`).
* Lembre-se de atualizar as importações, em `location.dart` e `loading_screen.dart`.
* Lembre-se também de que o método `getCurrentLocation` deve ser `async` assíncrono.

### Buscando dados de tempo via API REST

O que é uma API? API - **Application Programming Interface** ou Interface de Programação de Aplicação.
Trata-se de um conjunto de comandos, funções, protocolos e objetos que programadores podem usar para criar software ou interagir com um sistema externo (definição da Wikipedia).
Fornece comandos padrão para executar operações padrão, de forma a não precisar escrever código do zero.


Para buscar os dados, precisamos da biblioteca `http`. No arquivo `pubspec.yaml`, abaixo
da linha `geolocator`, acrescente: `http: ^0.13.5`. Não se esqueça de executar o pub get.
(canto superior direito da tela).

Agora, no arquivo `loading_screen.dart`, no começo do arquivo, adicione a linha que importa o pacote http:
`import 'package:http/http.dart' as http;`.

crie, dentro da classe `_LoadingScreenState` um método para baixarmos dados de exemplo da api do openweathermap:

```dart
void getData() async {
  var url = Uri.parse('https://samples.openweathermap.org/data/2.5/weather?lat=35&lon=139&appid=b6907d289e10d714a6e88b30761fae22');
  http.Response response = await http.get(url);

  if (response.statusCode == 200) { // se a requisição foi feita com sucesso
    var data = response.body;
    print(data);  // imprima o resultado
  } else {
    print(response.statusCode);  // senão, imprima o código de erro
  }
}
```

Neste momento, estamos apenas validando a resposta da consulta.
Para chamar o método, adicione a linha `getData()` ao método `build` da classe.

