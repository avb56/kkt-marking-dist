# kkt-marking

Маркировка в кассе — **ПМСР «ККТ Сайт Маркировка»**: разрешительный режим через
модуль **ТС ПИоТ**, проверка кодов в ККТ, код в чеке (`imcParams`, тег 1260),
база проданных кодов. Работает вместе с [kkt-core](https://github.com/avb56/kkt-core-dist):
ККТ — только через него.

Под этим именем и версией модуль представляется ТС ПИоТ (`client_info.name`,
`client_info.version`): программа, прошедшая проверку функциональной
совместимости у оператора, — это файл выпуска, а не приложение, в которое он
встроен. Контрольные суммы файла — в `SHA256SUMS` и `STREEBOG256SUMS`
(ГОСТ Р 34.11-2012, 256 бит).

Код обфусцирован. Файл подключается **как есть**: его нельзя пропускать через
сборщик или минификатор — после любой переделки он перестаёт запускаться.

| файл | для чего |
| --- | --- |
| `kkt-marking.mjs` | ES-модуль (es2020): браузер (Chrome 109 и новее) и Node 20+ |
| `SHA256SUMS`, `STREEBOG256SUMS` | контрольные суммы |

## Подключение

Как kkt-core: файл рядом со страницей, модулем; nginx не знает `.mjs` — проще
положить его под именем `kkt-marking.js`.

```js
const { KKT } = await import('/vendor/kkt-core.js');
const oMarking = await import('/vendor/kkt-marking.js');
oMarking.configure({
  log,                                      // { info, warn, error }(текст) — журнал; коды маркировки в него не пишутся
  oClient: { id: '…', token: '…' },         // из реестра оператора после проверки
});
```

## Сессия

```js
const oSession = oMarking.oCreateMarkingSession({
  fKktOf: rnm => oKKT,                      // ККТ из kkt-core
  oSoldStore,                               // { pGet, pSet, pDelete, pAll } — база проданных; по умолчанию в памяти
  pAlert: ({ sTitle, sText }) => …,         // сообщение кассиру
  pAskAmount: ({ sProduct, oKind, bReturn }) => …, // часть упаковки, мерный товар: количество или null
  pOfferPrice: ({ sProduct, sReason, nOfferPrice }) => …, // цена ниже минимальной: продать по ней?
  pDecide: ({ sProduct, sReason, bDoubt }) => …,   // ККТ отказала или не ответила: продать всё же?
  fNotice: (sLevel, sText) => …,            // 'info' | 'warning' | 'error'
  pNoTariff: ({ sProduct, sReason, sNeed }) => …, // маркировки нет в тарифе: предложить подключить (нет — fNotice)
  fDropItem: (rnm, oItem) => …,             // убрать позицию из чека; true — была
  fBusy: sText => …,                        // идёт проверка (null — кончилась)
  fCheckInBackground: () => true,           // проверять в ККТ сразу или перед печатью
});

const oScan = oMarking.oParseScan(sFromScanner);
const { sStatus } = await oSession.pScan({ rnm, oProduct, oScan, aItems, bKkt: true, oPiot, oOperator, fAdd });
// перед печатью
const { bReady, sText } = await oSession.pBeforePrint(rnm, () => aItems);
// в задание ККТ — для каждой позиции
Object.assign(oTaskItem, oSession.oFiscalFields(oItem));
// чек пробит
await oSession.pAfterReceipt(rnm, aItems, { bReturn, bKkt: true });
```

## Разрешение

Проверка кодов в ТС ПИоТ — по разрешению сервера политик ККТ Сайт на эту ККТ с
маркировкой в тарифе (`features.marking`), как и всё остальное в kkt-core (его
README, «Разрешение на работу»). Модуль проверяет разрешение сам: ККТ должна
быть экземпляром `KKT` из kkt-core, а модуль ТС ПИоТ — привязан оператором к
той же ККТ (заводской номер и ИНН из его `/info` сходятся с РНМ). Модуль ТС ПИоТ
на тестовом ФН (песочница оператора) работает без разрешения.

Маркировки нет в тарифе — отказ с `sNeed: 'marking'` (в итоге `pScan` и в
`pNoTariff`). С kkt-core 0.2.8 перед таким отказом модуль просит kkt-core
обменяться с сервером вне очереди: тариф, оплаченный только что, действует
сразу.

## Лицензия

Не MIT — `LICENSE`: в составе продуктов kktsite и у партнёров по договору.
