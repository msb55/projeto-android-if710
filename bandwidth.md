# Consumo de rede

### Cenários
Para o aplicativo de Podcast analisado, existem apenas dois possíveis cenários de consumo de rede do celular:
 - Ao inicializar, o feed dos Podcast é baixado, um arquivo em formato XML;
 - Ao usuário solicitar o download de um Podcast.

### Baixando o Feed
O arquivo XML que contém uma lista de itens - podcasts - é baixado assim que o usuário inicia o aplicativo com o intuito de sempre estar atualizado pelo feed. Por ser um arquivo XML, o seu tamanho é pequeno, mas de qualquer forma uma porção da banda larga é consumida.

Abaixo segue uma imagem do consumo de banda da aplicação, cada pico do gráfico significa o aplicativo sendo iniciado pelo usuario e realizando o download do feed:

![Imagem 1](https://raw.githubusercontent.com/msb55/projeto-android-if710/master/imagens_relatorio/sem_checagem.PNG)

É notável que o tempo de download é muito curto e a banda consumida também é pequena, mas em questões de internet móvel, qualquer consumo deve ser levado em consideração. Para um usuário que costuma usar bastante o aplicativo, e de tempos em tempos mantem o ciclo de abrir e fechar o aplicativo, esse pequeno consumo se torna cumulativo.

#### Melhorando o download do feed
Uma possível solução, e essa adotada para esse projeto, é armazenar quando foi a última atualização do feed, e apenas realizar o download quando uma nova "versão" estiver disponível. Mantendo a lógica de requisitar um novo feed ao abrir o aplicativo, modificando apenas essa condição.

Para isso utilizamos o campo ```"Last-Modified"``` do cabeçalho HTTP, podendo assim saber se a versão que temos armazenada no dispositivo é a mais recente ou não.

Para armazenar a última modificação do feed, utilizamos ```SharedPreferences``` como opção de armazenamento, uma vez que é apenas uma informação de formato String, que precisamos utilizar entre as sessões do usuário (mesmo que seja encerrado o processo do aplicativo).

Abaixo segue o código adicionado para poder verificar a última vez que o feed foi modificado:

```java
private static String FEED_LAST_MODIFIED = "LAST_MODIFIED"; // chave da preferencia compartilhada
```

```java
private boolean ifModified(String feed) {
        try {
            URL url = new URL(feed);
            HttpURLConnection conn = (HttpURLConnection) url.openConnection(); // conexao com a url do feed passada como parametro

            SharedPreferences prefs = getPreferences(0); // preferencia compartilhada em modo privado (apenas para a aplicacao)
            String last_modified = prefs.getString(FEED_LAST_MODIFIED, ""); // acessando com a chave

            if (!last_modified.isEmpty()) { // caso seja a primeira execucao do aplicativo, nada estara armazenado
                conn.setRequestProperty("If-Modified-Since", last_modified);
                if (conn.getResponseCode() == 304) { // status da requisicao HTTP para a propriedade informando que nao foi modificado [304 Not Modified]
                    return false; // responde que nao eh necessario o download do feed
                } else { // caso tenha sido, atualizar com a nova data
                    SharedPreferences.Editor editor = prefs.edit();
                    editor.putString(FEED_LAST_MODIFIED, conn.getHeaderField("Last-Modified"));
                    editor.commit();
                }
            } else { // apenas armazenar quando foi modificado
                SharedPreferences.Editor editor = prefs.edit();
                editor.putString(FEED_LAST_MODIFIED, conn.getHeaderField("Last-Modified"));
                editor.commit();
            }
        } catch (MalformedURLException e) {
            e.printStackTrace();
        } catch (IOException e) {
            e.printStackTrace();
        }
        return true;
    }
````

Como os comentários do código deixam claro, caso seja a primeira vez que o aplicativo é executado, a ```SharedPreference``` não terá alguma informação armazenada, assim apenas guarda o campo ```Last-Modified```. Caso não seja a primeira vez, verifica o código de resposta da requisição, caso seja 304, apenas informa que não é necessário realizar o download do feed, caso seja 200, armazena o novo resultado do campo ```Last-Modified``` e informa que é necessário realizar o download.

Com essa modificação o aplicativo retornou o seguinte resultado:

![Imagem 2](https://raw.githubusercontent.com/msb55/projeto-android-if710/master/imagens_relatorio/com_checagem.PNG)

O primerio pico é a primeira inicialização do aplicativo, onde não tem nada salvo ainda nas preferencias compartilhadas, assim é feito o download do feed. E é possível perceber algumas nuances após 14s, onde é o mesmo comportamento da primeira imagem: o aplicativo sendo fechado e iniciado, 3 vezes nesse caso.

Para ficar melhor a visualização das nuances mencionadas anteriormente, a imagem abaixo traz esse comportamento. É notável pela escala do gráfico a diferença, antes os 3 picos chegavam perto de 126KB/s, agora ficam perto de 2KB/s.

![Imagem 3](https://raw.githubusercontent.com/msb55/projeto-android-if710/master/imagens_relatorio/com_checagem2.PNG)

### Baixando o Podcast
Fica disponível ao usuário o download do episódio de Podcast, caso ele tenha interesse, e após o download estar completo, o mesmo é armazenado no dispositivo para que não seja necessário realizar o download novamente do mesmo episódio.

A única abordagem proposta para esse cenário é a verificação do tipo de internet disponível (Wi-Fi ou móvel) e desabilitar o download de um episódio caso o usuário esteja usando internet móvel. Nas configurações do aplicativo tornar possível o usuário escolher mesmo não estando com Wi-Fi conectado, apenas móvel, realizar o download.

Dessa forma previne que faça o consumo indesejado da banda, caso o usuário faça o download de um episódio sem ter noção de que está utilizando internet móvel.

#### Verificando o download do Podcast
Para verificar o tipo de conexão serviço da plataforma ```ConnectivityManager``` para coletar informações a respeito da conexão com a internet, e validando com a configuração mencionada anteriormente, negando o download do episódio caso o usuário esteja em conexão móvel e nas configurações indique que não é permitido.

Abaixo segue o código da verificação:
```java
private boolean verifyNetworkType() {
        Context context = getContext();

        SharedPreferences sharedPref = PreferenceManager.getDefaultSharedPreferences(context);
        boolean movel = sharedPref.getBoolean("downloadmovel", false); // preferencia compartilhada localizada nas configurações

        ConnectivityManager cm =  (ConnectivityManager) context.getSystemService(Context.CONNECTIVITY_SERVICE);
        NetworkInfo activeNetwork = cm.getActiveNetworkInfo(); // informacao acerca da conexão com a internet

        // libera o download quando a conexão é wifi, ou quando em mobile e permitido nas configurações
        return   activeNetwork.getTypeName().equalsIgnoreCase("wifi") ||
                (activeNetwork.getTypeName().equalsIgnoreCase("mobile") && movel);
    }
```