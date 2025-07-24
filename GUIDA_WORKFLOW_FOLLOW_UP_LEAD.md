# Guida Completa: Workflow di Follow-up Lead con n8n

## Panoramica

Questo workflow automatizza il processo di follow-up dei lead attraverso email e messaggi WhatsApp, utilizzando Perplexity AI per generare contenuti personalizzati. Il sistema gestisce automaticamente i lead dal database MySQL e invia messaggi di follow-up programmati.

## Architettura del Workflow

### Flusso Principale

```
Trigger → Ottieni Leads → Controlla Stato → [Cold Email/Follow-up] → Aggiorna Database
```

### Moduli Principali

1. **Trigger** - Avvia il workflow alle 9:00 ogni giorno
2. **Ottieni Leads** - Recupera i lead dal database MySQL
3. **Controlla Stato** - Determina se inviare cold email o follow-up
4. **Generazione Contenuti AI** - Utilizza Perplexity AI per creare messaggi personalizzati
5. **Invio Comunicazioni** - Email e WhatsApp
6. **Gestione Database** - Aggiorna stati e date di follow-up

## Configurazione Iniziale

### 1. Credenziali Necessarie

#### SMTP (Email)
- **Provider**: Qualsiasi provider SMTP (Gmail, Outlook, etc.)
- **Configurazione**:
  - Server SMTP
  - Porta (587 per TLS, 465 per SSL)
  - Username e password
  - Autenticazione abilitata

#### Twilio (WhatsApp)
- **Account SID**: Dal dashboard Twilio
- **Auth Token**: Dal dashboard Twilio
- **Numero WhatsApp**: Numero verificato su Twilio

#### MySQL
- **Host**: Indirizzo del server MySQL
- **Porta**: 3306 (default)
- **Database**: Nome del database
- **Username e Password**: Credenziali di accesso

#### Perplexity AI
- **API Key**: Chiave API di Perplexity
- **Endpoint**: https://api.perplexity.ai/chat/completions

### 2. Struttura Database

**IMPORTANTE**: Devi creare la tua tabella MySQL personalizzata. Ecco un esempio di struttura consigliata:

```sql
CREATE TABLE LEADS (
    id INT AUTO_INCREMENT PRIMARY KEY,
    Email VARCHAR(255) NOT NULL,
    Stato ENUM('Nuovo', 'Email Inviata', 'Risposta Ricevuta') DEFAULT 'Nuovo',
    Data_Prossimo_Follow_up DATE,
    Risposta_Ricevuta ENUM('Si', 'No') DEFAULT 'No',
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);
```

**Nota**: Adatta la struttura della tabella alle tue esigenze specifiche. I nomi delle colonne e la struttura possono variare in base ai tuoi requisiti.

## Dettaglio Moduli

### 1. Trigger (Schedule Trigger)
- **Tipo**: Schedule Trigger
- **Configurazione**: Esecuzione giornaliera alle 9:00
- **Scopo**: Avvia automaticamente il workflow

### 2. Ottieni Leads (MySQL)
- **Query**: `YOUR_SELECT_QUERY_HERE` (da personalizzare)
- **Scopo**: Recupera i lead da processare
- **Esempio**: `SELECT * FROM LEADS WHERE Stato IN ('Nuovo', 'Email Inviata')`

### 3. Controlla Stato (Switch)
- **Condizioni**:
  - **Caso 1**: Stato = "Nuovo" → Cold Email
  - **Caso 2**: Stato = "Email Inviata" → Follow-up

### 4. Perplexity API - Cold Email
- **Metodo**: POST
- **URL**: https://api.perplexity.ai/chat/completions
- **Headers**:
  ```
  Authorization: Bearer YOUR_PERPLEXITY_API_KEY
  Content-Type: application/json
  ```
- **Body**:
  ```json
  {
    "model": "llama-3.1-sonar-small-128k-online",
    "messages": [
      {
        "role": "system",
        "content": "Sei un esperto di marketing che scrive email persuasive per lead generation."
      },
      {
        "role": "user",
        "content": "Genera una cold email per un lead che si occupa di [settore]. L'email deve essere personalizzata, professionale e focalizzata sul valore che possiamo offrire."
      }
    ]
  }
  ```

### 5. Invia Cold Email (Email Send)
- **From**: Il tuo indirizzo email
- **To**: `={{ $('Leggi i lead').item.json.Email }}`
- **Subject**: "Come risparmiare il 25% del tempo con l'AI"
- **Content**: `={{ $json.message.content }}`

### 6. Cold Message (Twilio WhatsApp)
- **From**: Il tuo numero Twilio
- **To**: Numero del lead (se disponibile)
- **Message**: Messaggio di follow-up personalizzato

### 7. Genera Data Prossimo Follow Up (Set)
- **Output**: Data corrente + 5 giorni
- **Formato**: YYYY-MM-DD

### 8. Inserisci Followup (MySQL)
- **Query**: `YOUR_UPDATE_QUERY_HERE` (da personalizzare)
- **Esempio**: `UPDATE LEADS SET Stato = 'Email Inviata', Data_Prossimo_Follow_up = ? WHERE id = ?`

### 9. Controlla se è la Data per il Prossimo Follow Up (IF)
- **Condizione**: Data corrente = Data_Prossimo_Follow_up
- **True**: Procedi con follow-up
- **False**: Stop

### 10. Controlla se Abbiamo Ricevuto Risposta (IF)
- **Condizione**: Risposta_Ricevuta = "No"
- **True**: Invia follow-up
- **False**: Rimuovi lead

### 11. Perplexity API - Follow Up
- **Configurazione**: Simile al cold email ma con prompt diverso
- **Prompt**: "Genera un messaggio di follow-up gentile e non invadente per un lead che non ha ancora risposto."
- **Body**: Stesso formato del cold email ma con prompt di follow-up

### 12. Invia Email Follow Up (Email Send)
- **Subject**: "RE: Come risparmiare il 25% del tuo tempo con l'AI"
- **Content**: Contenuto generato da Perplexity AI

### 13. Follow Up Message (Twilio WhatsApp)
- **Message**: Messaggio di follow-up via WhatsApp

### 14. Rimuovi Lead (MySQL)
- **Query**: `YOUR_DELETE_QUERY_HERE` (da personalizzare)
- **Esempio**: `DELETE FROM LEADS WHERE id = ?`
- **Scopo**: Rimuove lead che non hanno risposto dopo il follow-up

## Personalizzazione

### Perplexity AI
**IMPORTANTE**: Devi configurare Perplexity AI:

1. **Endpoint**: https://api.perplexity.ai/chat/completions
2. **API Key**: Ottieni la tua chiave API da Perplexity
3. **Headers**: Configura l'autenticazione con Bearer token
4. **Prompt**: Personalizza i prompt per cold email e follow-up secondo le tue esigenze

**Modelli Perplexity consigliati**:
- llama-3.1-sonar-small-128k-online (veloce e economico)
- llama-3.1-sonar-medium-128k-online (bilanciato)
- llama-3.1-sonar-large-128k-online (alta qualità)

### Query MySQL
**IMPORTANTE**: Devi personalizzare tutte le query MySQL nel workflow:

1. **Ottieni Leads**: Sostituisci `YOUR_SELECT_QUERY_HERE` con la tua query per recuperare i lead
2. **Inserisci Followup**: Sostituisci `YOUR_UPDATE_QUERY_HERE` con la tua query per aggiornare lo stato
3. **Rimuovi Lead**: Sostituisci `YOUR_DELETE_QUERY_HERE` con la tua query per rimuovere i lead

**Esempi di query personalizzate**:
```sql
-- Per recuperare lead
SELECT * FROM your_table_name WHERE status IN ('new', 'sent') AND next_followup_date <= CURDATE();

-- Per aggiornare stato
UPDATE your_table_name SET status = 'followup_sent', next_followup_date = ? WHERE id = ?;

-- Per rimuovere lead
DELETE FROM your_table_name WHERE id = ? AND status = 'no_response';
```

### Modifica Messaggi
1. **Email**: Modifica i campi `subject` e `text` nei nodi Email Send
2. **WhatsApp**: Modifica il campo `message` nei nodi Twilio
3. **AI Prompts**: Personalizza i prompt per Perplexity nei nodi HTTP Request

### Timing
- **Frequenza Trigger**: Modifica `triggerAtHour` nel nodo Schedule Trigger
- **Intervallo Follow-up**: Modifica il calcolo nel nodo "Genera Data Prossimo Follow Up"

### Condizioni
- **Stati Lead**: Aggiungi nuovi stati nel database e modifica le condizioni
- **Logica Business**: Personalizza le condizioni nei nodi IF e Switch

## Monitoraggio e Debug

### Log di Esecuzione
- Controlla i log di n8n per errori
- Verifica le risposte delle API
- Monitora le statistiche di invio

### Metriche da Tracciare
- Tasso di apertura email
- Tasso di risposta
- Conversione lead
- Performance AI

### Troubleshooting Comuni

#### Errori SMTP
- Verifica credenziali
- Controlla configurazione server
- Verifica limiti di invio

#### Errori Twilio
- Verifica credenziali API
- Controlla numero WhatsApp
- Verifica crediti account

#### Errori MySQL
- Verifica connessione database
- Controlla permessi utente
- Verifica struttura tabelle

#### Errori Perplexity
- Verifica API key
- Controlla limiti di utilizzo
- Verifica formato richieste
- Controlla endpoint e autenticazione

## Sicurezza e Best Practices

### Protezione Dati
- Non committare mai credenziali nel codice
- Usa variabili d'ambiente per le API key
- Cripta i dati sensibili nel database

### Compliance
- Rispetta GDPR per i dati personali
- Implementa opt-out per le email
- Mantieni log delle comunicazioni

### Scalabilità
- Implementa rate limiting
- Usa code per grandi volumi
- Monitora performance database

## Estensioni Possibili

### Integrazioni Aggiuntive
- **CRM**: Integrazione con Salesforce, HubSpot
- **Analytics**: Google Analytics, Mixpanel
- **Social Media**: LinkedIn, Facebook
- **Calendar**: Google Calendar, Outlook

### Funzionalità Avanzate
- **A/B Testing**: Test di diversi messaggi
- **Segmentazione**: Targeting basato su comportamento
- **Scoring**: Sistema di scoring lead
- **Automazione**: Workflow più complessi

## Conclusione

Questo workflow fornisce una base solida per l'automazione del follow-up lead. La combinazione di email, WhatsApp e AI permette di creare un sistema scalabile e personalizzato per la gestione dei lead.

### Prossimi Passi
1. Configura le credenziali
2. Testa il workflow con pochi lead
3. Monitora le performance
4. Ottimizza basandosi sui risultati
5. Scala gradualmente

---

**Nota**: Questo workflow è un template e richiede personalizzazione in base alle specifiche esigenze aziendali e alla compliance normativa applicabile. 