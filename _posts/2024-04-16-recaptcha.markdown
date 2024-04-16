---
layout: post
title: "Come configurare reCaptcha enterprise di GCP"
date: 2024-04-16 14:00:00 +0200
excerpt_separator: <!--more-->
---

recaptcha enterprise è un servizio di Google Cloud Platform (GCP) che fornisce una soluzione di sicurezza avanzata per
proteggere i siti web da attacchi di spam e bot. In questo articolo, vedremo come configurare reCaptcha enterprise di
GCP per proteggere il tuo sito web.

<!--more-->

In questa guida utilizzerò gcloud-cli per interagire con i servizi di GCP. Assicurati di aver installato gcloud-cli sul
tuo computer e di aver effettuato il login con il tuo account GCP.

### Creazione della Key del recaptcha e configurazione del service account

Utilizziamo il seguente comando per creare la nuova key del recaptcha:

```bash
$ gcloud recaptcha keys create --web --domains="my.app.test","localhost" --display-name="my new recaptcha KEY" --integration-type invisible
```

Se il comando è andato a buon fine possiamo vedere la key appena creata con il seguente comando:

```bash
$ gcloud recaptcha keys list
```

A questo punto dobbiamo creare un service account che attraverso le meccaniche di IAM abbia a disposizione i privilegi
necessari ad attuare la verifica dei captcha

```bash
$ gcloud iam service-accounts create myapp-recaptcha
```

Copiamo il nome del service account appena creato e assegnamo i permessi necessari per effettuare la verifica dei
captcha:

```bash
$ gcloud projects add-iam-policy-binding my-gcp-project --member="serviceAccount:myapp-recaptcha@<my-gcp-project>.iam.gserviceaccount.com" --role=roles/recaptchaenterprise.agent
```

Adesso dobbiamo generare un json con le credenziali del service account appena creato. Questo JSON sarà utilizzato per
autenticare il client che effettuerà la verifica dei captcha.

```bash
$ gcloud iam service-accounts keys create my-service-account.json --iam-account myapp-recaptcha@<my-gcp-project>.iam.gserviceaccount.com
```

Il file json verrà generato in locale e avrà un contenuto simile a questo:

```
{
  "type": "service_account",
  "project_id": "my-project-id",
  "private_key_id": "---",
  "private_key": "-----BEGIN PRIVATE KEY-----SECRET SECRET SECRET\n-----END PRIVATE KEY-----\n",
  "client_email": "myapp-recaptcha@<my-gcp-project>.iam.gserviceaccount.com",
  "client_id": "---",
  "auth_uri": "https://accounts.google.com/o/oauth2/auth",
  "token_uri": "https://oauth2.googleapis.com/token",
  "auth_provider_x509_cert_url": "https://www.googleapis.com/oauth2/v1/certs",
  "client_x509_cert_url": "https://www.googleapis.com/robot/v1/metadata/x509/something...",
  "universe_domain": "googleapis.com"
}
```

Il processo che utilizzerà il file json per autenticarsi con il servizio di recaptcha enterprise dovrà avere una
variabile d'ambiente che punta al file json appena creato. Google Cloud SDK cerca la variabile d'ambiente
GOOGLE_APPLICATION_CREDENTIALS che deve contenere il path dove sarà copiato il file
json che abbiamo appena creato.

```
GOOGLE_APPLICATION_CREDENTIALS=google_keys/my-service-account.json.json
```

### Creare una piccola pagina di prova per generare i token

Se andiamo a cercare la nostra key dentro la console di google cloud, possiamo vedere che è stata creata con successo.
Inoltre possiamo anche accedere ad una area in cui sono presenti dei suggerimenti di integrazione via codice con vari
linguaggi.

La parte di frontend è molto semplice, basta includere il codice javascript fornito da google nella nostra pagina html.

```html

<html>
<head>

    <script src="https://www.google.com/recaptcha/enterprise.js?render=my-recaptcha-key"></script>
    <!-- Replace the variables below. -->
    <script>
        function onSubmit(token) {
            console.log(token);  // token is the generated token obtained from the client
        }
    </script>
</head>
<body>

<button class="g-recaptcha"
        data-sitekey="my-recaptcha-key"
        data-callback='onSubmit'
        data-action='submit'>
    Submit
</button>

</body>
</html>
```

Se servirete la pagina web su localhost noterete che è possibile generare il token senza problemi. Basta aprire la
console e copiarlo dopo aver premuto il pulsante di submit.

Noterete anche che la classe g-recaptcha fa comparire in fondo a destra della pagina un badge di google che indica che
il sito è protetto da reCaptcha. Se invece compare un errore vuol dire che la key non è stata configurata correttamente.

Durante la creazione di questo esempio ho notato che il badge di google non compare se il sito è servito su un dominio
diverso da quelli indicati in fase di creazione della key. Potete aggiungere altri domini alla key utilizzando la
console web o modificando la key utilizzando gcloud-cli. (non ho però riportato i comandi per modificare la key con la
shell perché non ho avuto bisogno di farlo)

### Verificare il challenge via backend

A questo punto utilizzando il proprio linguaggio preferito è possibile verificare il challenge del captcha.

Generate un token utilizzando la pagina web e poi utilizzate il token per verificare il challenge. Tenete anche conto
della action che avete definito sul pulsante!

Di seguito incollo un esempio in Java e springboot.

```java
package it.andreanicola.recaptchaenterprise
;
import com.google.cloud.recaptchaenterprise.v1.RecaptchaEnterpriseServiceClient;
import com.google.recaptchaenterprise.v1.Assessment;
import com.google.recaptchaenterprise.v1.CreateAssessmentRequest;
import com.google.recaptchaenterprise.v1.Event;
import com.google.recaptchaenterprise.v1.ProjectName;
import lombok.extern.slf4j.Slf4j;
import org.springframework.beans.factory.annotation.Value;
import org.springframework.stereotype.Service;

import java.io.IOException;

@Slf4j
@Service
public class ReCaptchaEnterpriseService {

    @Value("${google.recaptcha.key}")
    static private String projectID = "my-gcp-project-id";

    @Value("${google.recaptcha.project-id}")
    private String recaptchaKey;


    /**
     * Create an assessment to analyze the risk of a UI action.
     *
     * @param token  : The generated token obtained from the client.
     * @param action : The action the client used to create the token
     */
    public boolean assessRisk(String token, String action) throws IOException {

        // Per configurare questo client la env var GOOGLE_APPLICATION_CREDENTIALS
        // deve essere valorizzata con il path del file di credenziali.
        try (RecaptchaEnterpriseServiceClient client = RecaptchaEnterpriseServiceClient.create()) {

            // Set the properties of the event to be tracked.
            Event event = Event.newBuilder().setSiteKey(recaptchaKey).setToken(token).build();

            // Build the assessment request.
            CreateAssessmentRequest createAssessmentRequest = CreateAssessmentRequest.newBuilder().setParent(ProjectName.of(projectID).toString()).setAssessment(Assessment.newBuilder().setEvent(event).build()).build();
            Assessment response = client.createAssessment(createAssessmentRequest);

            log.info("reCAPTCHA assessment response {}", response);

            // Check if the token is valid.
            if (!response.getTokenProperties().getValid()) {
                log.info("The CreateAssessment call failed because the token was: {}", response.getTokenProperties().getInvalidReason().name());
                return false;
            }

            // Check if the expected action was executed.
            if (!response.getTokenProperties().getAction().equals(action)) {
                log.info("The action attribute in the reCAPTCHA tag ({}) does not match the action ({}) you are expecting to score", response.getTokenProperties().getAction(), action);
                return false;
            }

            float recaptchaScore = response.getRiskAnalysis().getScore();
            return recaptchaScore >= 0.9;

        }
    }
}
```

### Valutazione del rischio

Nel mio caso d'uso ho semplificato molto la valutazione del rischio. In un caso reale è possibile che si debba tenere
conto molti più fattori per determinare se il challenge è stato superato con successo.

Fate riferimento alla [documentazione](https://cloud.google.com/recaptcha-enterprise/docs/interpret-assessment-website)
ufficiale di Google per avere una panoramica più dettagliata su come valutare il rischio.

### Conclusione

Configurare questo tipo di captcha invisibile non è risultato molto complesso. La parte più fastidiosa è stata la
creazione del service account e la configurazione delle credenziali per l'autenticazione.

Spero che questa guida ti sia stata utile e ti abbia dato una panoramica generale su come configurare reCaptcha.
