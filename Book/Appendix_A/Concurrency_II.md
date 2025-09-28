عالی 👌
بذار ترجمه دقیق و حرفه‌ای 📝 این بخش از **ضمیمه A: Concurrency II** رو برات انجام بدم.👇

---

## 📎 ضمیمه A

# 🧵 هم‌روندی (Concurrency) – بخش دوم

✍️ نوشته: **Brett L. Schuchert**

این ضمیمه به فصل «هم‌روندی» در صفحه 177 کتاب 🧼 *Clean Code* کمک می‌کند و آن را گسترش می‌دهد. این بخش به‌صورت مجموعه‌ای از موضوعات مستقل نوشته شده است و معمولاً می‌توان آن‌ها را به هر ترتیبی مطالعه کرد. برای اینکه این نوع مطالعه امکان‌پذیر باشد، در بخش‌های مختلف مقداری تکرار دیده می‌شود.

---

## 💻 مثال Client/Server (کاربر / سرور)

فرض کنید یک برنامه ساده‌ی **Client/Server** داریم.
🔸 یک **سرور (Server)** روی یک *سوکت (Socket)* منتظر می‌ماند تا یک **کلاینت (Client)** به آن وصل شود.
🔸 یک **کلاینت** وصل شده و یک درخواست ارسال می‌کند.

---

## 🖥️ سرور (The Server)

در ادامه نسخه‌ی ساده‌شده‌ای از یک برنامه‌ی سرور را می‌بینید.
📄 نسخه کامل کد این مثال در صفحه 343 با عنوان **Client/Server Nonthreaded** آمده است:

```java
ServerSocket serverSocket = new ServerSocket(8009);
while (keepProcessing) {
    try {
        Socket socket = serverSocket.accept();
        process(socket);
    } catch (Exception e) {
        handle(e);
    }
}
```

این برنامه ساده منتظر یک اتصال می‌ماند، پیام ورودی را پردازش می‌کند و سپس دوباره منتظر درخواست بعدی از کلاینت می‌نشیند.
در ادامه، کد مربوط به **کلاینت** را می‌بینید که به این سرور وصل می‌شود:

```java
private void connectSendReceive(int i) {
    try {
        Socket socket = new Socket("localhost", PORT);
        MessageUtils.sendMessage(socket, Integer.toString(i));
        MessageUtils.getMessage(socket);
        socket.close();
    } catch (Exception e) {
        e.printStackTrace();
    }
}
```

---

## 📊 بررسی عملکرد (Performance)

عملکرد این جفت Client/Server چطور است؟
چطور می‌توانیم این عملکرد را به‌صورت رسمی توصیف کنیم؟
در اینجا یک تست وجود دارد که بررسی می‌کند آیا عملکرد سیستم «قابل قبول» است یا نه:

```java
@Test(timeout = 10000)
public void shouldRunInUnder10Seconds() throws Exception {
    Thread[] threads = createThreads();
    startAllThreadsw(threads);
    waitForAllThreadsToFinish(threads);
}
```

📌 تنظیمات اولیه این تست برای سادگی حذف شده است (به فایل **ClientTest.java** در صفحه 344 مراجعه کنید).
این تست انتظار دارد که اجرای کل عملیات در کمتر از **۱۰٬۰۰۰ میلی‌ثانیه** (۱۰ ثانیه) انجام شود ⏱️

---

## 🚀 بررسی توان عملیاتی (Throughput)

این یک مثال کلاسیک از اعتبارسنجی **Throughput** سیستم است.
در این سیستم، باید مجموعه‌ای از درخواست‌های کلاینت در ۱۰ ثانیه تکمیل شوند.
تا زمانی که سرور بتواند هر درخواست را به‌موقع پردازش کند، تست با موفقیت پاس می‌شود ✅

اما اگر تست شکست بخورد چه؟ 🤔
در صورتی که نوعی **Event Polling Loop** (حلقه بررسی رویدادها) طراحی نکنیم، در یک **تک‌رشته (Single Thread)** کار خاصی نمی‌توان برای سریع‌تر کردن این کد انجام داد.

آیا استفاده از چند رشته (Multiple Threads) می‌تواند مشکل را حل کند؟
شاید، اما ابتدا باید بدانیم زمان دقیقاً کجا صرف می‌شود. دو حالت کلی وجود دارد:

* 🕓 **I/O** (ورودی/خروجی): مثل استفاده از سوکت، اتصال به دیتابیس، انتظار برای Swap شدن حافظه مجازی و موارد مشابه.
* 🧮 **Processor** (پردازنده): مثل انجام محاسبات عددی، پردازش Regular Expressionها، Garbage Collection و موارد مشابه.

به‌طور معمول سیستم‌ها مقداری از هر دو مورد را دارند، اما در هر عملیات یکی از آن‌ها غالب است.

اگر کد **Processor-bound** باشد، افزودن سخت‌افزار پردازشی بیشتر می‌تواند Throughput را افزایش دهد و باعث موفقیت تست شود.
اما چون چرخه‌های CPU محدودند، افزودن Thread به یک مسئله Processor-bound باعث افزایش سرعت نمی‌شود ⚠️

از طرف دیگر، اگر فرآیند **I/O-bound** باشد، استفاده از هم‌روندی (Concurrency) می‌تواند بازده را افزایش دهد ✅
در این حالت، زمانی که یک بخش از سیستم منتظر I/O است، بخش دیگر می‌تواند از همان زمان برای پردازش کار دیگری استفاده کند و از CPU بهره‌وری بهتری بگیرد 💡

---

## 🧵 افزودن چندریسمانی (Adding Threading)

فرض کنیم تست عملکرد شکست خورده است.
چطور می‌توانیم **Throughput** را بهبود دهیم تا تست با موفقیت پاس شود؟

اگر متد `process` سرور **I/O-bound** باشد، یک روش این است که سرور را به شکل چندریسمانی (Threaded) تغییر دهیم.
کافی است فقط نحوه‌ی پردازش پیام را عوض کنیم:

```java
void process(final Socket socket) {
    if (socket == null)
        return;
    Runnable clientHandler = new Runnable() {
        public void run() {
            try {
                String message = MessageUtils.getMessage(socket);
                MessageUtils.sendMessage(socket, "Processed: " + message);
                closeIgnoringException(socket);
            } catch (Exception e) {
                e.printStackTrace();
            }
        }
    };
    Thread clientConnection = new Thread(clientHandler);
    clientConnection.start();
}
```

فرض کنید این تغییر باعث شده تست عملکرد پاس شود ✅
کد کامل شده است، درست است؟ 🤔

(👈 پاسخ این سؤال در ادامه فصل بررسی می‌شود…)

---

✅ ترجمه کامل و دقیق این بخش انجام شد؛ تمام اصطلاحات تخصصی مثل `Socket`, `Thread`, `I/O`, `Processor-bound` حفظ شده‌اند و متن برای خواننده بدون پیش‌زمینه برنامه‌نویسی روان و قابل فهم بازنویسی شده است.
از ایموجی‌ها هم به شکل استاندارد و مرتبط استفاده شده 📝✨

می‌خوای همین روند رو برای بخش بعدی (صفحه 320 به بعد) هم ادامه بدم؟ 📘👉
