# Bidding Service — спецификация сервиса ставок

## 1. Описание, границы и зависимости Bidding Service 

`Bidding Service` — это выделенный из монолита компонент, ранее известный как `Auction Engine`. Это самый критичный по latency узел всей системы: именно здесь взвешиваются ставки и определяется победитель аукциона.

Стратегия выделения из монолита — Strangler Fig: монолит не переписывается, а критичный путь постепенно "обрастает" новым сервисом, трафик на который переключается инкрементально (feature flag / процент трафика).

Ad Server остаётся ответственным за приём HTTP/OpenRTB-запроса, идентификацию пользователя и подбор кандидатов (это не входит в границы Bidding Service). Bidding Service получает уже подготовленный список кандидатов и решает, кто выигрывает и по какой цене.

То есть в Bidding Service происходит:
- Применение бизнес-правил взвешивания ставок (bid weighting rules).
- Определение победителя аукциона и цены.
- Чтение ставок и кампаний из реплики данных в локальном кэше Redis (это read-model).
- Публикация события результата аукциона (`AuctionEvent`) в Kafka — асинхронно, вне hot path.

`AuctionEvent` далее используется через Kafka в
- Delivery Service — для отложенной генерации HTML/JS-разметки баннера.
- Statistics/Analytics — для хранения исторических данных для отчётности.

Bidding Service не имеет прямого доступа к Postgres. Это ключевое отличие от текущего Auction Engine, который делает синхронные вызовы к БД и является главным источником деградации latency.

Все это позволит разделить критичные и некритичные потоки — Bidding Service ничего не пишет синхронно и ни от чего, кроме локального кэша, синхронно не читает.

Требования к самому сервису:
- Stateless — любой инстанс может обработать любой запрос, состояние (кэш конфигурации) вынесено в Redis.
- Горизонтальное масштабирование за счёт добавления инстансов (для выполнения требования "возможность масштабирования наиболее нагруженных компонентов").
- Отдельный deployment/pool ресурсов от остального монолита, чтобы нагрузка на Statistics/Analytics не влияла на Bidding Service (изоляция критичного пути).


## 2. API для Bidding Service

Сервис не имеет публичного REST API — он внутренний, вызывается только Ad Server-ом, поэтому используется gRPC.

### gRPC-контракт (`bidding.proto`, эскиз)

```protobuf
syntax = "proto3";
package adscale.bidding.v1;

service BiddingService {
  // Основной вызов hot path. Deadline выставляется вызывающей стороной (Ad Server), обычно 30–40 ms.
  rpc EvaluateAuction(AuctionRequest) returns (AuctionResponse);

  // Используется Gateway/оркестратором для readiness/liveness и для решения Circuit Breaker.
  rpc HealthCheck(HealthRequest) returns (HealthResponse);
}

message AuctionRequest {
  string request_id       = 1;  // сквозной ID запроса, для трассировки и AuctionEvent
  repeated AdCandidate candidates = 2;
  UserContext context      = 3;
}

message AdCandidate {
  string ad_id        = 1;
  string campaign_id  = 2;
  double bid_price    = 3;  // ставка, заявленная кандидатом (из конфигурации кампании)
}

message UserContext {
  string user_segment = 1;
  string geo          = 2;
  string device_type   = 3;
}

message AuctionResponse {
  string request_id      = 1;
  string winner_ad_id      = 2;
  string winner_campaign_id = 3;
  double price   = 4;
}
```
