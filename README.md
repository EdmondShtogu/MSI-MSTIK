# Krijimi i një Serveri OAuth në PHP

## Prezantimi

Nëse keni integruar ndonjëherë me një API tjetër që kërkon siguri (si Twitter), ndoshta keni konsumuar një shërbim OAuth.

[OAuth 2 është një authorization framework](https://github.com/EdmondShtogu/MSI-MSTIK/blob/master/Nj%C3%AB%20Hyrje%20n%C3%AB%20OAuth%202.pdf) që u mundëson aplikacioneve të marrin qasje të kufizuar në llogaritë e përdoruesve në një shërbim HTTP. Funksionon duke deleguar autentikimin e përdoruesit tek shërbimi që mban llogarinë e përdoruesit dhe autorizon aplikacionet e palëve të treta për të hyrë në llogarinë e përdoruesit. OAuth 2 siguron rrjedhat e autorizimit për aplikacionet në internet dhe desktop, dhe pajisjet mobile.

Kur keni të bëni me OAuth, zakonisht do ta shihni atë të zbatuar si një server OAuth me _dy-këmbë_ ose me _tre-këmbë_. Dallimi kryesor midis tyre është se autentifikimi me _dy-këmbë_ nuk përfshin një përdorues tjetër. Për shembull, nëse dëshironi të keni qasje në një informacion të një përdoruesi të caktuar në Twitter, do të konsumoni serverin me _tre-këmbë_, sepse duhet të gjenerohet një token qasje për përdoruesin në aplikacionin tuaj, përkundrejt vetëm Twitter që ju ofron ju një token. Ne do të përqëndrohemi në varietetin me _tre-këmbë_ pasi është më praktike për përdorim në botën reale.

Ne do të përdorim [oauth-php](http://code.google.com/p/oauth-php/) si framework për serverin tonë OAuth 2. Libraria është e organizuar në Google Code dhe nuk është e listuar në [Packagist](https://packagist.org/), por mund të instalohet duke përdorur [Composer](https://www.sitepoint.com/re-introducing-composer/). Për detaje, shikoni skedarin [composer.json](https://github.com/EdmondShtogu/MSI-MSTIK/blob/master/composer.json).

Le të fillojmë me rolet e OAuth!

OAuth përcakton katër role:

- Zotërues Burimesh (Resource Owner)
- Klient (Client)
- Server Burimesh (Resource Server)
- Server Autorizimi (Authorization Server)

## Kuptimi i rrjedhës

Kur përdorni një server OAuth me _tri-këmbë_, rrjedha tipike që një zhvillues do të ndërmarrë për të zbatuar dhe konsumuar shërbimin është si më poshtë:

![alt text](https://github.com/EdmondShtogu/MSI-MSTIK/blob/master/as.jpg)

Këtu është një shpjegim më i hollësishëm i hapave në diagramë:

1. Aplikacioni kërkon autorizimin për të përdorur burimet e shërbimit nga përdoruesi
2. Nëse përdoruesi autorizon kërkesën, aplikacioni merr autorizim
3. Aplikacioni kërkon një access token nga serveri i autorizimit (API) duke paraqitur vërtetimin e identitetit të vet dhe dhënien e autorizimit.
4. Nëse identiteti i aplikacionit vërtetohet dhe e drejta e autorizimit është e vlefshme, serveri i autorizimit (API) dërgon një access token te aplikacioni. Autorizimi është kryer.
5. Aplikacioni kërkon burime nga serveri i burimeve (API) dhe paraqet access token-in për autentikim
6. Nëse access token-i është i vlefshëm, serveri i burimeve (API) i shërben burimet aplikacionit

Tani le të fillojme me krijimin e serverit!

## Vendosja e bazës së të dhënave

Me librarinë oauth-php përmes kodit që gjëndet në library/store/mysql/mysql.sql, duhet të krijohet dhe inicializohet një bazë e re e të dhënave.

Nëse shfletoni nëpër tabela, do të shihni se tabela oauth\_server\_registry përmban një fushë të quajtur osr\_usa\_id\_ref.

Kjo do të popullohet gjatë procesit të regjistrimit nga serveri OAuth. Nëse ju keni një tabelë tashmë ekzistuese të përdoruesve për të cilën do të lidhet do të ishte pë ishte perfekt! Por nëse jo, atëherë këtu është një script SQL për të krijuar një tabelë standarde të përdoruesit:

    CREATE TABLE users (
        id INTEGER UNSIGNED NOT NULL AUTO_INCREMENT,
        name VARCHAR(255) NOT NULL DEFAULT '',
        password VARCHAR(255) NOT NULL DEFAULT '',
        email VARCHAR(255) NOT NULL DEFAULT '', 
        created DATE NOT NULL DEFAULT '0000-00-00',

        PRIMARY KEY (id)
    );

## Krijimi i serverit OAuth

Le të fillojmë të shkruajmë serverin OAuth. Kodi i më poshtë është i përgjithshëm për pjesën tjetër të kodit tonë, kështu që le ta vendosim atë në një skedarin të veçantë include/common.php:

     <?php
      require_once '../vendor/autoload.php';

      session_start();

      // Add a header indicating this is an OAuth server
      header('X-XRDS-Location: http://' . $_SERVER['SERVER_NAME'] .
        '/services.xrds.php');

      // Connect to database
      $db = new PDO('mysql:host=localhost;dbname=oauth', 'dbuser', 'dbpassword');

      // Create a new instance of OAuthStore and OAuthServer
      $store = OAuthStore::instance('PDO', array('conn' => $db));
      $server = new OAuthServer();

Skedari shton një HTTP header për çdo kërkesë për të informuar klientët se ky është një server OAuth. Vini re se referencat services.xrds.php; ky skedar ofrohet me shembullin që vjen me librarinë oauth-php. Ju duhet ta kopjoni atë nga example/server/www/services.xrds.php në direktorinë root publike të web serverit tuaj.

Linjat e ardhshme të kodit krijojnë një lidhje me bazën e të dhënave (informacionin e lidhjes duhet të përditësohen në përputhje me konfigurimin tuaj) dhe krijon instanca të objekteve të rinj OAuthStore  dhe OAuthServer  të ofruara nga libraria.

Konfigurimi për serverin OAuth tani është i përfunduar dhe serveri është i gatshëm të implementohet plotësisht. Në shembujt e ardhshëm, linja includes/common.php  duhet të përfshihet çdo herë për të inicizuar serverin.

## Lejimi i Regjistrimit

Para se zhvilluesit të mund të konsumojnë serverin tuaj OAuth, ata duhet të regjistrojnë veten me të. Për të lejuar këtë, ne duhet të krijojmë një formular themelor regjistrimi. Fushat e mëposhtme janë të nevojshme për shkak se ato do të kalohen në librari: requester\_name  dhe requester\_email. Fushat e mbetura janë fakultative: application\_uri dhe callback\_uri.

      <form method="post" action="register.php">
       <fieldset>
        <legend>Register</legend>
        <div>
         <label for="requester_name">Name</label>
         <input type="text" id="requester_name" name="requester_name">
        </div>
        <div>
         <label for="requester_email">Email</label>
         <input type="text" id="requester_email" name="requester_email">
        </div>
        <div>
         <label for="application_uri">URI</label>
         <input type="text" id="application_uri" name="application_uri">
        </div>
        <div>
         <label for="callback_uri">Callback URI</label>
         <input type="text" id="callback_uri" name="callback_uri">
        </div>
       </fieldset>
       <input type="submit" value="Register">
      </form>

Siç e përmendëm më herët, libraria supozon se keni përdorues ekzistues të cilët dëshirojnë të konsumojnë shërbimin tuaj. Në kodin e mëposhtëm krijojmë një përdorues të ri në tabelën e përdoruesve, më pas marrim ID-në dhe e kalojmë atë në metodën updateConsumer () duke krijuar (ose përditësuar) çelësin e konsumatorit dhe sekretin për këtë përdorues. Kur ju ta integroni këtë në aplikacionin tuaj, kjo pjesë duhet të modifikohet dhe të vendoset nën procesin e hyrjes ku tashmë e dini se kush është përdoruesi i cili po regjistrohet për qasje.

      <?php
      $stmt = $db->prepare('INSERT INTO users (name, email, created) ' . 'VALUES (:name, :email, NOW())');
      $stmt->execute(array('name' => $_POST['requester_name'], => $_POST['requester_email']));
      $id = $db->lastInsertId();
      $key = $store->updateConsumer($_POST, $id, true);
      $c = $store->getConsumer($key, $id);
      ?>
      <p><strong>Save these values!</strong></p>
      <p>Consumer key: <strong><?=$c['consumer_key']; ?></strong></p>
      <p>Consumer secret: <strong><?=$c['consumer_secret']; ?></strong></p>

Pas përfundimit të regjistrimit, do të kthehen si output çelësi i ri i konsumatorit dhe çelësi sekret i tij. Këto vlera duhet të ruhen nga përdoruesi për tu përdorur në të ardhmen.

Tani që një përdorues është regjistruar, ai mund të fillojë të bëjë kërkë për një token qasjeje!

## Gjenerimi i Request Token-it

Pasi një përdorues të jetë regjistruar, duhet të kryehet një kërkesë OAuth në skedarin request\_token.php. Ky skedar (edhe një herë për shkak të librarisë) është shumë i thjeshtë:

      <?php
      require_once 'include/oauth.php';
      $server->requestToken();

Metoda requestToken() kujdeset për vërtetimin se përdoruesi ka një çelës të vlefshëm dhe nënshkrimin. Nëse kërkesa është e vlefshme, një request token e re kthehet.

## Shkëmbimi i Request Token për një Access Token

Përdoruesi duhet të ridrejtohet në faqen tuaj të identifikimit sapo të krijohet një request token. Kjo faqe duhet të presë parametrat URL të mëposhtëm:  oauth\_token  dhe oauth\_callback.

Faqja hyrëse duhet të marrë përdoruesin nga tabela e përdoruesve. Pasi të shikohet, ID e përdoruesit kalon (së bashku me oauth\_token) me metodën authorizeVerify() të ofruar nga libraria. Duke supozuar se përdoruesi ka autorizuar aplikacionin, ID-ja e përdoruesit të kyçur më pas shoqërohet me çelësin e konsumatorit duke u lejuar atyre qasje të sigurtë në të dhënat e këtij përdoruesi.

Logjika e nevojshme e një login.php  bazë mund të duket si më poshtë:

      <?php
      $sql = 'SELECT id FROM users WHERE email = :email';
      $stmt = $db->prepare($sql);
      $result = $stmt->exec(array('email' => $_POST['requester_email']));
      $row = $result->fetch(PDO::FETCH_ASSOC);
      if (!$row) {}// incorrect login
      $id = $row['id'];
      $result->closeCursor();
      $rs = $server->authorizeVerify();
      $authorized = array_key_exists('allow', $_POST);
      $server->authorizeFinish($authorized, $id);

Pas regjistrimit të përdoruesit, do të ridrejtohet përsëri në faqen e internetit të zhvilluesit (duke përdorur parametrin oauth\_callback ) me një token të vlefshme. Ky token dhe verifikimi i çelësit më pas mund të përdoret në shkëmbimin për një token të vlefshëm të qasjes.

      <?php
      require_once 'include/oauth.php';
      
      $server->accessToken();

Ky skedar është po aq i thjeshtë sa kërkesa e krijuar më parë nga request\_token.php. Puna bëhet brenda metodës accessToken() të ofruar nga libraria oauth-php. Me një kërkesë të suksesshme, një oauth\_token  dhe oauth\_token\_secret  të vlefshme kthehen që duhet të ruhen dhe të përdoren në kërkesat e ardhshme në API-n tuaj.

## Validimi i një kërkese

Në këtë pikë, serveri OAuth është në përdorim. Por ende ne duhet të verifikojmë që një kërkesë përmban një nënshkrimit të vlefshëm OAuth. Më poshtë jepet një skedar test bazë që bën këtë:

      <?php
      require_once 'includes/oauth.php';
      if (OAuthRequestVerifier::requestIsSigned()) {
          try {
              $req = new OAuthRequestVerifier(); $id = $req->verify();
              if ($id) { echo 'Hello ' . $id; }} 
          catch (OAuthException $e) {
              header('HTTP/1.1 401 Unauthorized');
              header('WWW-Authenticate: OAuth realm=""');
              header('Content-Type: text/plain; charset=utf8');
              echo $e->getMessage(); exit(); }}
      

Në shembull, nëse kërkesa vërtetohet, thjesht kthejmë një përgjigje me ID-in e përdoruesit të që hyri. Është e sugjerueshme të krijohet një metode e ri-përdorshme që përmban këtë kod për çdo thirrje API që kërkon siguri.

## Testimi i Serverit OAuth

Së fundi, është koha për të testuar serverin OAuth. Më poshtë është një skedar i thjeshtë që kryen hapat e mësipërm për të kërkuar që një përdorues të identifikohet dhe të kryejë një kërkesë të sigurt:

      <?php
      define('OAUTH_HOST', 'http://' . $_SERVER['SERVER_NAME']);
      $id = 1;
      // Init the OAuthStore
      $options = array(
          'consumer_key' => '<MYCONSUMERKEY>',
          'consumer_secret' => '<MYCONSUMERSECRET>',
          'server_uri' => OAUTH_HOST,
          'request_token_uri' => OAUTH_HOST . '/request_token.php',
          'authorize_uri' => OAUTH_HOST . '/login.php',
          'access_token_uri' => OAUTH_HOST . '/access_token.php'
      );
      OAuthStore::instance('Session', $options);
      if (empty($_GET['oauth_token'])) {
          // get a request token
          $tokenResultParams = OauthRequester::requestRequestToken($options['consumer_key'], $id);
          header('Location: ' . $options['authorize_uri'] .
              '?oauth_token=' . $tokenResultParams['token'] . 
              '&oauth_callback=' . urlencode('http://' .
                  $_SERVER['SERVER_NAME'] . $_SERVER['PHP_SELF']));
      }
      else {
          // get an access token
          $oauthToken = $_GET['oauth_token'];
          $tokenResultParams = $_GET;
          OAuthRequester::requestAccessToken($options['consumer_key'],
              $tokenResultParams['oauth_token'], $id, 'POST', $_GET);
          $request = new OAuthRequester(OAUTH_HOST . '/test_request.php',
              'GET', $tokenResultParams);
          $result = $request->doRequest(0);
          if ($result['code'] == 200) {
              var_dump($result['body']);
          }
          else {
              echo 'Error';
          }
      }


OAuth i bashkangjit kohëzgjatjen dhe nënshkrimin çdo kërkese, dhe përsëri kjo librari do ta kryejë këtë për ne.

## Përmbledhje

Në këtë pikë, ju duhet të dini se si të krijoni një server OAuth. Duke përdorur skedarin test\_request.php si shembull, mund të filloni të krijoni më shumë veçori që sigurohen me Oauth!
