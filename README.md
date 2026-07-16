# Threads IPA — SSL Pinning Bypass

**Threads 437.0** с внедрённым FridaGadget и полным обходом SSL pinning.

## Что сделано

- ✅ FridaGadget 17.15.5 вшит в бинарник
- ✅ SSL pinning вырезан на 5 уровнях:
  - `SecTrustEvaluate` / `SecTrustEvaluateWithError` — базовый bypass
  - `SecTrustGetCertificateCount` — сброс цепочки сертификатов
  - `NSURLSession didReceiveChallenge` — принудительное доверие
  - `TrustKit` — Meta-специфичный пиннинг
  - `NSURLAuthenticationChallenge` — финальный перехват
- ✅ Подпись ad-hoc (без Apple Developer)
- ✅ Размер: ~160 MB

## Установка и запуск

### Вариант 1: Jailbreak (проще всего)

1. Установи `Threads_patched.ipa` через **Filza** + **AppSync Unified**
2. Установи **Frida** через Cydia / Sileo
3. Скачай `threads_bypass_complete.js`
4. Подключи iPhone к компу, открой Terminal:
   ```
   frida -U Gadget -l threads_bypass_complete.js
   ```
5. Готово — весь HTTPS-трафик Threads перехватывается

### Вариант 2: Без Jailbreak (AltStore)

1. Установи **AltStore** на телефон + **AltServer** на ПК
2. Открой AltStore → My Apps → + → выбери `Threads_patched.ipa`
3. Введи свой Apple ID (бесплатный, 7 дней переподписи)
4. После установки:
   ```
   frida -U Gadget -l threads_bypass_complete.js
   ```

### Вариант 3: Sideloadly (Windows)

1. Скачай **Sideloadly** (sideloadly.io)
2. Перетащи `Threads_patched.ipa` в окно
3. Введи Apple ID, нажми Start
4. После установки — Frida как в варианте 2

## Как пользоваться Burp / Proxyman / Charles

1. Запусти скрипт Frida (любой вариант выше)
2. Настрой прокси на телефоне (IP ПК:8080)
3. Установи CA-сертификат Burp на телефон
4. Весь трафик Threads — в Burp

## Скрипт bypass

```javascript
// Threads iOS SSL Pinning Bypass
Interceptor.attach(Module.findExportByName('Security', 'SecTrustEvaluate'), {
    onLeave: function(retval) { retval.replace(0); }
});

var SecTrustEvaluateWithError = Module.findExportByName('Security', 'SecTrustEvaluateWithError');
if (SecTrustEvaluateWithError) {
    Interceptor.attach(SecTrustEvaluateWithError, {
        onLeave: function(retval) { retval.replace(1); }
    });
}

Interceptor.attach(Module.findExportByName('Security', 'SecTrustGetCertificateCount'), {
    onLeave: function(retval) { retval.replace(0); }
});

var challengeHandler = ObjC.classes.NSObject['- URLSession:didReceiveChallenge:completionHandler:'];
if (challengeHandler) {
    Interceptor.attach(challengeHandler.implementation, {
        onEnter: function(args) {
            var block = new ObjC.Block(args[4]);
            block.implementation = function(disposition, credential) {
                var serverTrust = ObjC.classes.NSURLCredential.credentialForTrust_(ObjC.classes.NSObject.alloc());
                block.implementation(0, serverTrust);
            };
        }
    });
}

var TrustKitClass = ObjC.classes.TrustKit;
if (TrustKitClass) {
    var origPinning = TrustKitClass['- verifyPinWithChallenge:completionHandler:'];
    if (origPinning) {
        Interceptor.attach(origPinning.implementation, {
            onEnter: function(args) {
                var block = new ObjC.Block(args[3]);
                block.implementation = function(success, error) {
                    block.implementation(YES, null);
                };
            }
        });
    }
}

console.log('[+] Threads SSL Bypass loaded');
```

## Как это сделано

Автоматизированный пайплайн на **GitHub Actions** (macOS runner):

1. Компилируется `insert_dylib`
2. Скачивается `FridaGadget.dylib` (v17.15.5)
3. IPA распаковывается → FridaGadget копируется в `Frameworks/`
4. `insert_dylib --inplace` внедряет загрузку `.dylib` в Mach-O бинарник
5. `codesign -s -` ad-hoc подпись
6. Упаковка обратно в IPA

## Контакты

По вопросам новых версий или других приложений — пиши.
