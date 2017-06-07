---
layout:     post
title:      Testy integracyjne RabbitMQ przy pomocy Docker'a w Spring
date:       2017-06-07
summary:    
categories: rabbitmq, spring, java, programowanie
---

W tym artykule przedstawię jeden ze sposobów na testowanie kodu odpowiedzialnego za konsumpcję wiadomości przechowywanych w RabbitMQ.
Do implementacji takiego testu posłużę się JUnitem. Jeżeli chodzi o setup środowiska pod testy integracyjne użyję systemu kontenerów Docker.

## Obiekt do testów

Klasa, która konsumuje wiadomości prezentuje się następująco:

{% highlight java lineanchors %}

@Service
public class Consumer implements MessageListener {
    private EmailService emailService;
    private ObjectMapper objectMapper;
    private CounterService counterService;
    public static final String METRIC_EMAIL_FAIL = "email.fail";

    public Consumer(
        EmailService emailService, 
        ObjectMapper objectMapper, 
        CounterService counterService
    ) {
        this.emailService = emailService;
        this.objectMapper = objectMapper;
        this.counterService = counterService;
    }

    @Override
    public void onMessage(Message message) {
        EmailRequestAmqpMessage emailRequest = null;

        try {
            emailRequest = objectMapper.readValue(message.getBody(), EmailRequestAmqpMessage.class);
        } catch (IOException e) {
            counterService.increment(METRIC_EMAIL_FAIL);
            return;
        }

        emailService.send(emailRequest.email, emailRequest.message);
    }
}

{% endhighlight %}

Jak można wywnioskować po nazwie jest to klasa pełniąca rolę konsumenta wpiętego do RabbitMQ. Klasa implementuje interface `MessageListener`, który wymusza implementację metody `onMessage` co sprawia, że klasa `Consumer` nie stanowi POJO a wpisuje się w kontekst messagingu.

## Przypadki testowe

Klasa testowa będzie posiadała dwa przypadki:
* przypadek pozytywny kiedy klasa `Consumer` jest w stanie poprawnie wykonać mapowanie danych wejściowych na obiekt `EmailRequestAmqpMessage` a później wysyła informację do `CounterService` o poprawnej wysyłce maila. 
* przypadek negatywny w którym klasa `Consumer` obsługuje wyjątek wynikający z niepoprawnego mapowania danych wejściowych na obiekt `EmailRequestAmqpMessage`
