# 📎 ضمیمه A

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


## 📝 مشاهدات مربوط به سرور (Server Observations) 🖥️

نسخه به‌روزشده‌ی سرور، تست عملکرد را در کمی بیش از **۱ ثانیه** با موفقیت به پایان می‌رساند ⏱️✅
اما متأسفانه این راه‌حل کمی ساده‌انگارانه است و مشکلات جدیدی را به همراه می‌آورد ⚠️

❓ **سرور ما ممکن است چند تا Thread ایجاد کند؟**
کدی که نوشتیم هیچ محدودیتی برای تعداد Threadها تعیین نکرده است. بنابراین ممکن است به سقف تعداد Threadهایی برسیم که **JVM (Java Virtual Machine)** اجازه می‌دهد.
برای بسیاری از سیستم‌های ساده، این مقدار کافی است.
اما اگر سیستم ما قرار باشد از کاربران زیادی در اینترنت عمومی پشتیبانی کند، چه؟ 🌐
اگر تعداد زیادی کاربر هم‌زمان متصل شوند، ممکن است سیستم عملاً از کار بیفتد یا به شدت کند شود 🐢💥

فعلاً مشکلات رفتاری را کنار بگذاریم. راه‌حل ارائه‌شده، از نظر **پاکی (Cleanliness)** و **ساختار (Structure)** نیز مشکلاتی دارد.
❗ کد سرور چه تعداد «مسئولیت» دارد؟

* 🔸 مدیریت اتصال‌های Socket
* 🔸 پردازش کلاینت
* 🔸 سیاست Threading (Threading Policy)
* 🔸 سیاست خاموش کردن سرور (Server Shutdown Policy)

متأسفانه همه‌ی این مسئولیت‌ها داخل تابع `process` قرار دارند.
علاوه بر آن، این کد سطوح مختلفی از **انتزاع (Abstraction)** را با هم مخلوط می‌کند.
بنابراین با وجود کوچک بودن تابع `process`، لازم است این کد دوباره ساختاربندی شود ✂️📌

> 📌 (می‌توانید خودتان با بررسی کد قبل و بعد از تغییر این موضوع را ببینید.
> کد بدون Thread از صفحه 343 شروع می‌شود و نسخه Threaded از صفحه 346.)

---

## 🧭 اصل تک‌مسئولیتی (Single Responsibility Principle)

سرور به دلایل متعددی نیاز به تغییر دارد؛ بنابراین این کد اصل **Single Responsibility Principle (SRP)** را نقض می‌کند ❌

برای حفظ تمیزی سیستم‌های هم‌روند (Concurrent Systems)، مدیریت Threadها باید فقط در چند مکان محدود و کاملاً کنترل‌شده انجام شود ✅
علاوه بر این، هر بخشی از کد که مسئول Threadهاست باید فقط همین کار را انجام دهد و نه هیچ چیز دیگر.

چرا؟
زیرا پیدا کردن و رفع مشکلات هم‌روندی (Concurrency Bugs) به‌خودی‌خود کار بسیار دشواری است 😵‍💫
حالا تصور کنید مجبور باشید هم‌زمان با آن، مشکلات دیگری را هم رفع کنید! 😬

---

## 🧱 جداسازی مسئولیت‌ها

اگر برای هرکدام از مسئولیت‌های بالا — از جمله مدیریت Thread — یک کلاس جداگانه بسازیم، آنگاه با تغییر استراتژی مدیریت Thread، فقط بخش کوچکی از کد تغییر می‌کند و مسئولیت‌های دیگر آلوده نمی‌شوند 👌

این کار باعث می‌شود آزمایش (Test) بخش‌های دیگر نیز ساده‌تر شود چون نیازی به درگیر شدن با Threadها نخواهد بود 🧪✨

در ادامه نسخه‌ی به‌روزشده‌ای از سرور را می‌بینید که دقیقاً همین کار را انجام می‌دهد:

```java
public void run() {
  while (keepProcessing) {
    try {
      ClientConnection clientConnection = connectionManager.awaitClient();
      ClientRequestProcessor requestProcessor 
        = new ClientRequestProcessor(clientConnection);
      clientScheduler.schedule(requestProcessor);
    } catch (Exception e) {
      e.printStackTrace();
    }
  }
  connectionManager.shutdown();
}
```

در این نسخه، همه‌ی بخش‌های مربوط به Thread در یک مکان متمرکز شده‌اند:
👉 `clientScheduler`
اگر مشکلی در هم‌روندی پیش بیاید، فقط همین نقطه را باید بررسی کنیم 🕵️‍♂️

```java
public interface ClientScheduler {
    void schedule(ClientRequestProcessor requestProcessor);
}
```

---

## 🧭 سیاست فعلی (Current Policy)

پیاده‌سازی سیاست فعلی بسیار ساده است:

```java
public class ThreadPerRequestScheduler implements ClientScheduler {
    public void schedule(final ClientRequestProcessor requestProcessor) {
        Runnable runnable = new Runnable() {
            public void run() {
                requestProcessor.process();
            }
        };
        Thread thread = new Thread(runnable);
        thread.start();
    }
}
```

با جدا کردن کامل مدیریت Threadها در یک مکان، تغییر نحوه‌ی کنترل Threadها بسیار آسان‌تر می‌شود 🔄✨

مثلاً برای استفاده از **Java 5 Executor Framework** فقط کافی است یک کلاس جدید بنویسیم و آن را در سیستم قرار دهیم:

---

## 📄 Listing A-1 — `ExecutorClientScheduler.java` 🧵

```java
import java.util.concurrent.Executor;
import java.util.concurrent.Executors;

public class ExecutorClientScheduler implements ClientScheduler {
    Executor executor;
    public ExecutorClientScheduler(int availableThreads) {
        executor = Executors.newFixedThreadPool(availableThreads);
    }
    public void schedule(final ClientRequestProcessor requestProcessor) {
        Runnable runnable = new Runnable() {
            public void run() {
                requestProcessor.process();
            }
        };
        executor.execute(runnable);
    }
}
```

---

## 🟩 نتیجه‌گیری (Conclusion) ✅

افزودن هم‌روندی (Concurrency) در این مثال خاص، روشی برای **افزایش توان عملیاتی (Throughput)** سیستم را نشان می‌دهد و همچنین نحوه‌ی اعتبارسنجی این توان عملیاتی از طریق تست را آموزش می‌دهد 🧪📈

متمرکز کردن تمام کدهای هم‌روندی در چند کلاس مشخص، نمونه‌ای از به‌کارگیری اصل **Single Responsibility Principle** است.
در برنامه‌نویسی هم‌روند، این موضوع اهمیت ویژه‌ای دارد زیرا این نوع کدنویسی ذاتاً پیچیده است 🧠⚡

---

## 🧭 مسیرهای ممکن اجرای کد (Possible Paths of Execution)

بیایید متد زیر را بررسی کنیم 👇

```java
public class IdGenerator {
  int lastIdUsed;
  public int incrementValue() {
    return ++lastIdUsed;
  }
}
```

فرض کنید **Overflow عددی** را نادیده بگیریم و فقط یک Thread به یک نمونه‌ی واحد از `IdGenerator` دسترسی دارد.
در این حالت فقط یک مسیر اجرای ممکن و یک نتیجه‌ی تضمین‌شده وجود دارد ✅

* مقدار برگشتی برابر با مقدار `lastIdUsed` خواهد بود و هر دو مقدار **۱ واحد بیشتر از مقدار قبل از فراخوانی متد** هستند.

---

اما اگر دو Thread به‌طور هم‌زمان از این متد استفاده کنند و آن را تغییر ندهیم چه اتفاقی می‌افتد؟ 🤔
نتایج ممکن در صورتی که هر Thread فقط یک بار `incrementValue` را صدا بزند، چه خواهند بود؟
(فرض کنید مقدار اولیه‌ی `lastIdUsed` برابر با **93** باشد):

* 🧵 Thread 1 مقدار **94** می‌گیرد، Thread 2 مقدار **95** می‌گیرد، و مقدار نهایی `lastIdUsed` برابر با **95** می‌شود.
* 🧵 Thread 1 مقدار **95** می‌گیرد، Thread 2 مقدار **94** می‌گیرد، و مقدار نهایی `lastIdUsed` برابر با **95** می‌شود.
* 🧵 هر دو Thread مقدار **94** را می‌گیرند، و مقدار نهایی `lastIdUsed` برابر با **94** می‌شود ⚠️

نتیجه‌ی نهایی ممکن است **شگفت‌انگیز** باشد 😳 اما واقعاً امکان‌پذیر است.
برای درک دلیل وقوع این نتایج متفاوت، باید تعداد مسیرهای ممکن اجرای کد و نحوه‌ی اجرای آن توسط **JVM (Java Virtual Machine)** را درک کنیم 🧠🔍
عالی! حالا ترجمه دقیق و روان بخش جدید درباره **تعداد مسیرهای اجرای ممکن** و مفاهیم Atomic و Byte-code را برات آماده کردم 📝👇

---

## 🛤️ تعداد مسیرهای اجرای ممکن (Number of Paths) 🔄

برای محاسبه‌ی تعداد مسیرهای ممکن اجرای کد، ابتدا باید **Byte-code تولیدشده** را بررسی کنیم 💻

یک خط کد جاوا:

```java
return ++lastIdUsed;
```

به **هشت دستور Byte-code** تبدیل می‌شود.

دو Thread می‌توانند اجرای این هشت دستور را به‌گونه‌ای در هم آمیخته (Interleave) کنند، مثل این‌که یک کارت‌دهنده کارت‌ها را هنگام شلوف کردن کارت‌ها در هم می‌ریزد 🃏

حتی با فقط هشت دستور برای هر Thread، تعداد بسیار زیادی ترتیب مختلف (Shuffled Outcomes) ممکن است رخ دهد 😲

---

### 🔹 حالت ساده

برای **N دستور پشت سر هم**، بدون حلقه یا شرط، و **T Thread**، تعداد کل مسیرهای اجرای ممکن به صورت زیر است:

[
\text{تعداد مسیرهای ممکن} = \frac{(N*T)!}{(N!)^T}
]

---

### 🔹 محاسبه ترتیب‌های ممکن (Calculating the Possible Orderings)

این توضیح از ایمیل Uncle Bob به Brett آمده است 📧:

* با **N مرحله و T Thread**، مجموع مراحل برابر با (T \times N) است.
* قبل از هر مرحله یک **Context Switch** رخ می‌دهد که بین Threadها انتخاب می‌کند.
* بنابراین هر مسیر را می‌توان به صورت رشته‌ای از اعداد نشان داد که **نمایانگر Context Switchها** است.

#### مثال با دو مرحله (A و B) و دو Thread (1 و 2):

شش مسیر ممکن:

```
1122, 1212, 1221, 2112, 2121, 2211
```

یا به شکل مراحل:

```
A1B1A2B2, A1A2B1B2, A1A2B2B1, A2A1B1B2, A2A1B2B1, A2B2A1B1
```

برای **سه Thread**، ترتیب‌ها مشابه زیر هستند:

```
112233, 112323, 113223, ...
```

⚠️ نکته: در این رشته‌ها، همیشه باید N نمونه از هر Thread وجود داشته باشد.
برای مثال، رشته `111111` **غیرمعتبر** است زیرا هیچ نمونه‌ای از Threadهای 2 و 3 ندارد.

> ⚠️ این مدل کمی ساده‌سازی شده است، اما برای فهم موضوع کافی است.

---

### 🔹 تعداد مسیرهای غیرتکراری

برای مثال دو مرحله و دو Thread:

* رشته‌های چهار رقمی داریم: دو 1 و دو 2
* می‌توان 1ها یا 2ها را بدون تغییر معنا با هم جابجا کرد
* بنابراین **چهار ایزومورف (Isomorph)** برای هر رشته داریم، یعنی سه مسیر تکراری و یک مسیر غیرتکراری

[
4! \times 0.25 = 6
]

همین منطق در حالت‌های دیگر هم صادق است.

#### فرمول کلی:

[
\text{تعداد مسیرهای ممکن} = \frac{(T*N)!}{(N!)^T}
]

* برای (T = 2, N = 2) داریم (6 = 24/4)
* برای (T = 3, N = 2) داریم (90 = 720/8)
* برای (T = 3, N = 3) داریم (1680 = 9!/6^3)

برای مثال ما (یک خط جاوا = ۸ خط Byte-code و دو Thread)، **تعداد کل مسیرهای ممکن = 12,870**
اگر نوع `lastIdUsed` **long** باشد، هر Read/Write به دو عملیات تبدیل می‌شود و تعداد مسیرها به **2,704,156** می‌رسد.

---

### 🔹 افزودن `synchronized`

اگر متد را این‌طور تغییر دهیم:

```java
public synchronized void incrementValue() {
    ++lastIdUsed;
}
```

* تعداد مسیرهای ممکن برای دو Thread به **۲** کاهش می‌یابد
* در حالت کلی برابر با (N!) خواهد بود ✅

---

## 🔎 بررسی دقیق‌تر (Digging Deeper)

چرا دو Thread می‌توانستند قبل از افزودن `synchronized` همان مقدار را دریافت کنند؟ 🤔
ابتدا باید با مفهوم **عملیات اتمیک (Atomic Operation)** آشنا شویم.

### 🔹 عملیات اتمیک (Atomic Operation)

یک عملیات اتمیک، عملیاتی است که **قابل قطع شدن نیست** 🚫✂️

مثال:

```java
01: public class Example {
02:    int lastId;
03:
04:    public void resetId() {
05:        value = 0;  // این خط اتمیک است
06:    }
07:
08:    public int getNextId() {
09:        ++value;    // این خط اتمیک نیست
10:    }
11:}
```

* در خط ۵، مقدار ۰ به `lastId` اختصاص داده می‌شود
* طبق **Java Memory Model**، اختصاص مقدار به یک متغیر ۳۲ بیتی **اتمیک است** ✅

❗ اگر نوع `lastId` را به `long` تغییر دهیم، خط ۵ دیگر اتمیک نخواهد بود.

* طبق JVM، اختصاص به یک مقدار ۶۴ بیتی نیاز به **دو عملیات ۳۲ بیتی** دارد
* در این فاصله، ممکن است Thread دیگری وارد شده و یکی از مقادیر را تغییر دهد ⚠️

---

### 🔹 عملگر پیش‌افزایشی `++` (Pre-increment)

* این عملگر **قابل قطع شدن است**
* بنابراین اتمیک نیست
* برای درک دقیق، باید **Byte-code** تولیدشده را بررسی کنیم 🔍

---

### 🔹 تعاریف مهم قبل از ادامه

1. **Frame** — هر فراخوانی متد یک Frame نیاز دارد.

   * شامل آدرس برگشت (Return Address)، پارامترها و متغیرهای محلی است.
   * این تکنیک برای تعریف **Call Stack** استفاده می‌شود و در زبان‌های مدرن برای فراخوانی تابع و بازگشتی ضروری است.

2. **Local Variable** — هر متغیری که در دامنه متد تعریف شده است.

   * تمام متدهای غیر استاتیک حداقل یک متغیر `this` دارند که به **شیء جاری** اشاره می‌کند، شیئی که پیام اخیر را دریافت کرده و باعث فراخوانی متد شده است.

3. **Operand Stack** — بیشتر دستورها در JVM پارامتر می‌گیرند.

   * Operand Stack جایی است که این پارامترها قرار می‌گیرند.
   * ساختار LIFO (Last-In, First-Out) دارد.


## 🧩 Byte-code متد resetId() و رفتار اتمیک آن 🔹

### Byte-code تولیدشده برای `resetId()`:

| Mnemonic        | Description                                                                                   | Operand Stack After |
| --------------- | --------------------------------------------------------------------------------------------- | ------------------- |
| ALOAD 0         | بارگذاری متغیر ۰ روی Operand Stack. متغیر ۰ همان `this` است؛ شیء جاری که پیام را دریافت کرده. | this                |
| ICONST_0        | قرار دادن مقدار ثابت ۰ روی Operand Stack                                                      | this, 0             |
| PUTFIELD lastId | ذخیره مقدار بالای Stack (۰) در فیلد lastId شیء جاری (`this`)                                  | <empty>             |

⚡ این سه دستور **اتمیک هستند**، یعنی حتی اگر Thread هنگام اجرای هر کدام متوقف شود، داده‌های مورد نیاز PUTFIELD توسط Thread دیگر قابل دسترسی نیستند.

* نتیجه: مقدار ۰ به طور مطمئن در فیلد ذخیره می‌شود ✅
* همه داده‌ها محلی هستند، بنابراین **تداخل بین Threadها وجود ندارد**.

💡 اگر این سه دستور توسط ده Thread همزمان اجرا شود، تعداد ترتیب‌های ممکن **4.38679733629e+24** خواهد بود، اما **تنها یک نتیجه نهایی** وجود دارد.

* حتی اگر نوع `lastId` از int به long تغییر کند، نتیجه همان خواهد بود زیرا همه Threadها مقدار ثابتی را اختصاص می‌دهند.

---

## ⚠️ مشکل `++` در getNextId()

Byte-code متد `getNextId()` (فرض مقدار اولیه `lastId = 42`):

| Mnemonic        | Description                              | Operand Stack After |
| --------------- | ---------------------------------------- | ------------------- |
| ALOAD 0         | بارگذاری `this` روی Operand Stack        | this                |
| DUP             | کپی بالای Stack                          | this, this          |
| GETFIELD lastId | گرفتن مقدار lastId و قرار دادن روی Stack | this, 42            |
| ICONST_1        | قراردادن ۱ روی Stack                     | this, 42, 1         |
| IADD            | جمع دو مقدار بالای Stack                 | this, 43            |
| DUP_X1          | کپی و قرار دادن قبل از this              | 43, this, 43        |
| PUTFIELD value  | ذخیره مقدار 43 در فیلد value شیء جاری    | 43                  |
| IRETURN         | بازگرداندن مقدار بالای Stack             | <empty>             |

⚠️ مثال تداخل Threadها:

* Thread اول بعد از GETFIELD متوقف می‌شود (`lastId = 42`)
* Thread دوم کل متد را اجرا کرده و مقدار را به 43 می‌رساند
* Thread اول ادامه می‌دهد و مقدار 42 روی Stack را +1 می‌کند → دوباره 43 می‌شود
* نتیجه: **یک افزایش از دست می‌رود** ✅

💡 با افزودن `synchronized` به متد `getNextId()`, این مشکل رفع می‌شود.

---

## 🔹 نتیجه‌گیری درباره Byte-code و Atomic

* نیاز به درک کامل Byte-code نیست
* کافی است بدانیم **Threadها می‌توانند روی هم تاثیر بگذارند**
* نکته مهم: ++ **اتمیک نیست**

برای برنامه‌نویسی همزمان باید بدانیم:

1. کجاها اشیاء یا مقادیر مشترک هستند
2. کدهایی که ممکن است باعث **خواندن/نوشتن همزمان** شوند
3. چگونه از بروز مشکلات همزمان جلوگیری کنیم ⚡

---

## 🏗️ آشنایی با کتابخانه‌ها (Knowing Your Library)

### Executor Framework

* از Java 5 به بعد، **Executor Framework** برای مدیریت پیشرفته Threadها ارائه شد
* بسته `java.util.concurrent` شامل این کلاس‌هاست
* استفاده از **Thread Pool** در این Framework باعث **کد تمیزتر، کوچک‌تر و قابل مدیریت‌تر** می‌شود

✅ ویژگی‌ها:

* مدیریت Threadها
* تغییر اندازه خودکار
* بازسازی Thread در صورت نیاز
* پشتیبانی از **Futures** برای پردازش همزمان

---

### مثال استفاده از Future

```java
public String processRequest(String message) throws Exception {
    Callable<String> makeExternalCall = new Callable<String>() {
        public String call() throws Exception {
            String result = "";
            // ارسال درخواست خارجی
            return result;
        }
    };
    Future<String> result = executorService.submit(makeExternalCall);
    String partialResult = doSomeLocalProcessing();
    return result.get() + partialResult;
}
```

* `makeExternalCall` اجرا می‌شود
* پردازش محلی ادامه می‌یابد
* `result.get()` **منتظر تکمیل Future** می‌ماند ⏳

---

## ⚡ راهکارهای Nonblocking

### مثال قدیمی با `synchronized`:

```java
public class ObjectWithValue {
    private int value;
    public synchronized void incrementValue() { ++value; }
    public int getValue() { return value; }
}
```

### نسخه جدید Nonblocking با AtomicInteger:

```java
public class ObjectWithValue {
    private AtomicInteger value = new AtomicInteger(0);
    public void incrementValue() {
        value.incrementAndGet();
    }
    public int getValue() {
        return value.get();
    }
}
```

* با استفاده از **Compare-And-Swap (CAS)**
* عملکرد معمولاً **بهتر از نسخه قدیمی** است
* فرض می‌کند **چند Thread به ندرت مقدار مشترک را تغییر می‌دهند**
* اگر مقدار تغییر نکرده باشد، تغییر انجام می‌شود؛ در غیر این صورت دوباره تلاش می‌کند 🔄

### شبیه‌سازی CAS:

```java
int variableBeingSet;
void simulateNonBlockingSet(int newValue) {
    int currentValue;
    do {
        currentValue = variableBeingSet;
    } while(currentValue != compareAndSwap(currentValue, newValue));
}
int synchronized compareAndSwap(int currentValue, int newValue) {
    if(variableBeingSet == currentValue) {
        variableBeingSet = newValue;
        return currentValue;
    }
    return variableBeingSet;    
}
```

* CAS **اتمیک است**
* بررسی می‌کند که مقدار هنوز همان مقدار قبلی است
* اگر بله → تغییر انجام می‌شود
* اگر نه → دوباره تلاش می‌کند تا موفق شود


## ⚠️ کلاس‌های غیرهمزمان (Nonthread-Safe Classes) 🔹

برخی کلاس‌ها ذاتاً **thread-safe نیستند**. مثال‌ها:

* `SimpleDateFormat`
* ارتباط با دیتابیس (Database Connections)
* Containers در `java.util`
* `Servlets`

📌 برخی Collectionها متدهای thread-safe دارند، اما هر عملیاتی که **چند متد را پشت سر هم فراخوانی کند** امن نیست.
مثال:

```java
if(!hashTable.containsKey(someKey)) {
    hashTable.put(someKey, new SomeValue());
}
```

* هر متد به تنهایی thread-safe است ✅
* اما Thread دیگر ممکن است **بین containsKey و put** مقداری اضافه کند ⚠️

---

## 🔐 روش‌های حل مشکل همزمانی در Collectionها

### ۱. قفل کردن از سمت کلاینت (Client-based Locking)

```java
synchronized(map) {
    if(!map.containsKey(key))
        map.put(key,value);
}
```

* همه کاربران باید همین الگو را رعایت کنند
* هر Thread قبل از استفاده از Map باید آن را قفل کند و بعد آزاد کند

### ۲. استفاده از Wrapper و Adapter (Server-based Locking)

```java
public class WrappedHashtable<K, V> {
    private Map<K, V> map = new Hashtable<K, V>();
    public synchronized void putIfAbsent(K key, V value) {
        if (!map.containsKey(key))
            map.put(key, value);
    }
}
```

### ۳. استفاده از Collectionهای thread-safe

```java
ConcurrentHashMap<Integer, String> map = new ConcurrentHashMap<>();
map.putIfAbsent(key, value);
```

* کلاس‌های `java.util.concurrent` متدهایی مانند `putIfAbsent()` ارائه می‌دهند 🔹

---

## 🔄 وابستگی بین متدها و مشکلات همزمانی

مثال ساده: یک `IntegerIterator` که متدهای `hasNext()`, `next()`, `getNextValue()` دارد.

```java
public class IntegerIterator implements Iterator<Integer> {
    private Integer nextValue = 0;

    public synchronized boolean hasNext() {
        return nextValue < 100000;
    }

    public synchronized Integer next() {
        if (nextValue == 100000)
            throw new IteratorPastEndException();
        return nextValue++;
    }

    public synchronized Integer getNextValue() {
        return nextValue;
    }
}
```

* اگر **یک Thread** این Iterator را اجرا کند، مشکلی نیست
* اما اگر **دو Thread مشترک داشته باشند**، احتمال رخ دادن **استثنا در آخرین عنصر** وجود دارد
* مشکل: Thread1 فراخوانی `hasNext()` کرده و قبل از `next()` متوقف می‌شود

  * Thread2 همان متدها را اجرا می‌کند و وضعیت تغییر می‌کند
  * Thread1 دوباره ادامه می‌دهد و **ممکن است از انتهای Iterator عبور کند** ⚠️

---

## 🔐 سه راه‌حل برای این مشکل

### ۱. تحمل خطا (Tolerate the Failure)

* ممکن است خطا **آسیبی نرساند**
* مثل مثال، کلاینت می‌تواند Exception را گرفته و پاکسازی کند
* ⚠️ این روش **کمی شلخته است**، شبیه پاک کردن Memory Leak با ریستارت شبانه

---

### ۲. قفل گذاری از سمت کلاینت (Client-Based Locking)

```java
IntegerIterator iterator = new IntegerIterator();
while (true) {
    int nextValue;
    synchronized (iterator) {
        if (!iterator.hasNext())
            break;
        nextValue = iterator.next();
    }
    doSomethingWith(nextValue);
}
```

* هر کلاینت با `synchronized` قفل ایجاد می‌کند
* این کار **تکرار کد** است و **اصل DRY را نقض می‌کند**
* اما برای ابزارهای غیر thread-safe شخص ثالث ضروری است ⚠️

💡 خطر: همه برنامه‌نویسان باید به **قفل‌گذاری صحیح** دقت کنند، در غیر این صورت مشکلات پنهان و دشوار برای ردیابی ایجاد می‌شود.

---

### ۳. قفل‌گذاری از سمت سرور (Server-Based Locking)

* سرور مسئول مدیریت دسترسی همزمان است
* کلاینت‌ها بدون تغییر کد اصلی می‌توانند از منابع به صورت امن استفاده کنند
* **راهکار بهتر و مطمئن‌تر نسبت به قفل‌گذاری از سمت کلاینت**

---

### 📖 داستان واقعی از قفل‌گذاری سمت کلاینت

* سیستم **Multi-terminal Time-Sharing** برای حسابداری یک اتحادیه
* استفاده از **Client-based Locking** برای منابع مشترک
* یک برنامه‌نویس فراموش می‌کند قفل‌گذاری کند
* نتیجه: یکی از ترمینال‌ها گاهی “lock up” می‌کند
* مشکل: **Ring-buffer counter** از هماهنگی با pointer خارج شده بود
* راه‌حل: یک **hack** برای ریست خودکار buffer
* تجربه: **Client-based Locking می‌تواند بسیار خطرناک باشد** ❌
* درس مهم: همیشه تا حد امکان **مدیریت همزمانی را به سرور بسپارید**

## قفل‌گذاری سمت سرور (Server-Based Locking) 🔒

تکرار کد را می‌توان با اعمال تغییرات زیر روی `IntegerIterator` حذف کرد:

```java
public class IntegerIteratorServerLocked {
    private Integer nextValue = 0;

    public synchronized Integer getNextOrNull() {
        if (nextValue < 100000)
            return nextValue++;
        else
            return null;
    }
}
```

و کد کلاینت هم تغییر می‌کند:

```java
while (true) {
    Integer nextValue = iterator.getNextOrNull();
    if (nextValue == null)
        break;
    // do something with nextValue
}
```

در این حالت، ما **API کلاس خود را برای آگاهی از چند Thread بودن تغییر داده‌ایم**. کلاینت باید بررسی کند که مقدار null نیست، به جای چک کردن `hasNext()`.

به طور کلی، باید **قفل‌گذاری سمت سرور** را ترجیح دهید به دلایل زیر:

* کاهش کد تکراری—قفل‌گذاری سمت کلاینت هر کلاینت را مجبور می‌کند تا سرور را درست قفل کند. با قرار دادن کد قفل در سرور، کلاینت‌ها آزاد هستند تا از شیء استفاده کنند بدون نگرانی از نوشتن کد قفل اضافی.
* افزایش کارایی—می‌توان یک سرور thread-safe را با یک سرور غیر thread-safe در حالت single-threaded جایگزین کرد و تمام overhead اضافی را حذف کرد.
* کاهش احتمال خطا—فقط کافی است یک برنامه‌نویس قفل را فراموش کند.
* اعمال یک سیاست واحد—سیاست در یک مکان است، سرور، به جای اینکه در چند مکان مختلف، هر کلاینت، باشد.
* کاهش دامنه متغیرهای مشترک—کلاینت از آنها و نحوه قفل شدن آنها آگاه نیست. همه چیز در سرور پنهان است. وقتی مشکلی پیش می‌آید، تعداد مکان‌های بررسی کمتر است.

اگر کد سرور را در اختیار ندارید:

* از **ADAPTER** برای تغییر API و اضافه کردن قفل استفاده کنید:

```java
public class ThreadSafeIntegerIterator {
    private IntegerIterator iterator = new IntegerIterator();

    public synchronized Integer getNextOrNull() {
        if(iterator.hasNext())
            return iterator.next();
        return null;
    }
}
```

* یا بهتر است، از **Collection‌های thread-safe با اینترفیس‌های گسترش یافته** استفاده کنید.

---

## افزایش Throughput ⚡

فرض کنید می‌خواهیم به اینترنت برویم و محتوای مجموعه‌ای از صفحات را از لیستی از URLها بخوانیم. با هر صفحه خوانده شده، آن را پردازش می‌کنیم تا آمار جمع‌آوری شود. بعد از خواندن همه صفحات، یک گزارش خلاصه چاپ می‌کنیم.

کلاس زیر محتوای یک صفحه را با توجه به URL بازمی‌گرداند:

```java
public class PageReader {
    //...
    public String getPageFor(String url) {
        HttpMethod method = new GetMethod(url);
        try {
            httpClient.executeMethod(method);
            String response = method.getResponseBodyAsString();
            return response;
        } catch (Exception e) {
            handle(e);
        } finally {
            method.releaseConnection();
        }
    }
}
```

کلاس بعدی، Iterator است که محتوای صفحات را بر اساس Iterator از URLها فراهم می‌کند:

```java
public class PageIterator {
    private PageReader reader;
    private URLIterator urls;

    public PageIterator(PageReader reader, URLIterator urls) {
        this.urls = urls;
        this.reader = reader;
    }

    public synchronized String getNextPageOrNull() {
        if (urls.hasNext())
            return getPageFor(urls.next());
        else
            return null;
    }

    public String getPageFor(String url) {
        return reader.getPageFor(url);
    }
}
```

یک نمونه از `PageIterator` می‌تواند بین چند Thread مختلف به اشتراک گذاشته شود، هر Thread از نمونه خودش از `PageReader` برای خواندن و پردازش صفحات استفاده می‌کند.

توجه کنید که **بلوک synchronized بسیار کوچک نگه داشته شده**. فقط بخش حیاتی داخل `PageIterator` است. همیشه بهتر است **کمترین قسمت ممکن را همگام‌سازی کنید** تا بیشترین قسمت را.

---

### محاسبه Throughput با Single-Thread

فرض کنید مقادیر زیر برای محاسبات ساده اعمال شود:

* زمان I/O برای دریافت یک صفحه (میانگین): ۱ ثانیه
* زمان پردازش برای تجزیه صفحه (میانگین): ۰.۵ ثانیه
* I/O هیچ CPU مصرف نمی‌کند، پردازش ۱۰۰٪ CPU نیاز دارد.

برای **N صفحه که توسط یک Thread پردازش می‌شوند**، زمان کل اجرا برابر است با:

```
1.5 ثانیه * N
```

شکل A-1 یک نمونه از ۱۳ صفحه را نشان می‌دهد که حدود ۱۹.۵ ثانیه طول می‌کشد.

<p align="center">
  <img src=../../assets/image/Appendix_A/img-A.1.jpeg/>
</p>

## محاسبه Throughput با چند Thread 🧵⚡

اگر امکان دارد صفحات را به هر ترتیبی دریافت کنیم و پردازش صفحات مستقل از هم باشد، می‌توان از **چند Thread** برای افزایش throughput استفاده کرد.

چه اتفاقی می‌افتد اگر از **سه Thread** استفاده کنیم؟ چند صفحه می‌توانیم در همان زمان دریافت کنیم؟

همان‌طور که در شکل A-2 مشاهده می‌کنید، **راه‌حل چند Threadه** اجازه می‌دهد که پردازش صفحات که وابسته به CPU است، با خواندن صفحات که وابسته به I/O است هم‌پوشانی داشته باشد.

در یک دنیای ایده‌آل، این یعنی **پردازنده به طور کامل مورد استفاده قرار می‌گیرد**. هر خواندن یک‌ثانیه‌ای صفحه با دو پردازش هم‌پوشانی دارد. بنابراین می‌توانیم **دو صفحه در ثانیه پردازش کنیم**، که **سه برابر throughput راه‌حل تک‌Threadه** است.

<p align="center">
  <img src=../../assets/image/Appendix_A/img-A.2.jpeg/>
</p>

## بن‌بست (Deadlock) ⚠️🔒

تصور کنید یک **وب‌اپلیکیشن** با دو **منبع مشترک محدود** داریم:

* یک **پول از اتصال‌های دیتابیس** برای ذخیره‌سازی کار در حال انجام محلی
* یک **پول از اتصال‌های MQ** به مخزن اصلی

فرض کنید دو عملیات در این اپلیکیشن وجود دارد: **Create** و **Update**

* **Create** — ابتدا اتصال به مخزن اصلی و دیتابیس گرفته می‌شود، سپس با سرویس مخزن اصلی صحبت می‌کند و بعد کار را در دیتابیس محلی ذخیره می‌کند.
* **Update** — ابتدا اتصال به دیتابیس و سپس به مخزن اصلی گرفته می‌شود، داده‌ها از دیتابیس محلی خوانده شده و به مخزن اصلی ارسال می‌شوند.

حالا چه اتفاقی می‌افتد وقتی تعداد کاربران بیشتر از اندازه هر **پول** باشد؟ فرض کنید هر پول اندازه‌اش ده است:

1. ده کاربر عملیات **Create** را اجرا می‌کنند، بنابراین **تمام ۱۰ اتصال دیتابیس گرفته می‌شود** و هر Thread بعد از گرفتن اتصال دیتابیس اما قبل از گرفتن اتصال به مخزن اصلی متوقف می‌شود.
2. ده کاربر عملیات **Update** را اجرا می‌کنند، بنابراین **تمام ۱۰ اتصال مخزن اصلی گرفته می‌شود** و هر Thread بعد از گرفتن اتصال مخزن اصلی اما قبل از گرفتن اتصال دیتابیس متوقف می‌شود.
3. حالا ده Thread مربوط به **Create** باید منتظر اتصال به مخزن اصلی بمانند، و ده Thread مربوط به **Update** باید منتظر اتصال دیتابیس باشند.
4. **بن‌بست!** سیستم دیگر هرگز بازیابی نمی‌شود.

ممکن است این سناریو غیرواقعی به نظر برسد، اما چه کسی سیستم را می‌خواهد که هر هفته یک‌بار کاملاً قفل شود؟ چه کسی می‌خواهد سیستم را با علائم غیرقابل بازتولید اشکال‌زدایی کند؟ چنین مشکلاتی در محیط واقعی اتفاق می‌افتند و هفته‌ها طول می‌کشد تا حل شوند.

یک «راه‌حل» معمول، اضافه کردن دستورات **debugging** است تا بفهمیم چه اتفاقی می‌افتد. البته، کد debug آنقدر سیستم را تغییر می‌دهد که بن‌بست در شرایط دیگر رخ دهد و ماه‌ها بعد دوباره اتفاق بیفتد.

### شرایط لازم برای وقوع بن‌بست:

۱. **انحصار متقابل (Mutual Exclusion)**
زمانی که چند Thread نیاز دارند از یک منبع مشترک استفاده کنند و این منابع:

* نمی‌توانند همزمان توسط چند Thread استفاده شوند
* تعدادشان محدود است
  مثال معمول: اتصال دیتابیس، فایل باز برای نوشتن، قفل رکورد یا Semaphore

۲. **قفل و انتظار (Lock & Wait)**
وقتی یک Thread منبعی را گرفت، آن را آزاد نمی‌کند تا تمام منابع دیگر مورد نیازش را نگرفته و کارش را کامل نکرده باشد.

۳. **عدم پیش‌دستی (No Preemption)**
یک Thread نمی‌تواند منابع Thread دیگر را بگیرد. تنها راه دسترسی Thread دیگر، آزاد کردن منبع توسط Thread فعلی است.

۴. **انتظار دایره‌ای (Circular Wait)**
این وضعیت به «در آغوش مرگ» هم معروف است.
تصور کنید دو Thread، **T1 و T2** و دو منبع **R1 و R2** داریم.

* T1، R1 را دارد
* T2، R2 را دارد
* T1 همچنین به R2 نیاز دارد
* T2 همچنین به R1 نیاز دارد

این وضعیت شبیه شکل **A-3** است:
<p align="center">
  <img src=../../assets/image/Appendix_A/img-A.3.jpeg/>
</p>

تمام این چهار شرط باید برقرار باشند تا بن‌بست اتفاق بیفتد. اگر حتی یکی از این شرایط شکسته شود، بن‌بست غیرممکن است.

### شکستن شرط Mutual Exclusion

یکی از راه‌ها برای جلوگیری از بن‌بست، دور زدن شرط **انحصار متقابل** است. می‌توان این کار را با:

* استفاده از منابعی که اجازه استفاده همزمان می‌دهند، مثل **AtomicInteger**
* افزایش تعداد منابع تا برابر یا بیشتر از تعداد Threadهای رقابتی شود
* بررسی اینکه تمام منابع آزاد هستند قبل از گرفتن هر کدام

متأسفانه اکثر منابع محدود هستند و اجازه استفاده همزمان نمی‌دهند. گاهی هم هویت دومین منبع به نتیجه عملیات روی منبع اول بستگی دارد.

### شکستن شرط Lock & Wait

راه دیگر این است که از انتظار خودداری کنیم. قبل از گرفتن هر منبع آن را بررسی کرده و اگر مشغول بود، تمام منابع گرفته شده را آزاد کرده و دوباره شروع کنیم.

این روش مشکلاتی دارد:

* **Starvation** — یک Thread ممکن است همیشه نتواند منابع مورد نیازش را بگیرد.
* **Livelock** — چند Thread ممکن است همزمان یک منبع را بگیرند و آزاد کنند، بارها و بارها، مخصوصاً با الگوریتم‌های ساده زمان‌بندی CPU.

اگرچه این روش ناکارآمد به نظر می‌رسد، بهتر از هیچ است و تقریباً همیشه قابل پیاده‌سازی است.

### شکستن شرط No Preemption

راه دیگر، اجازه دادن به Threadها برای گرفتن منابع از Threadهای دیگر است. معمولاً از یک مکانیزم درخواست ساده استفاده می‌شود. وقتی یک Thread می‌بیند منبع مشغول است، از مالک آن می‌خواهد آزادش کند. اگر مالک هم منتظر منبع دیگری باشد، همه منابع را آزاد کرده و دوباره شروع می‌کند.

این مشابه روش قبلی است اما مزیت دارد که Thread می‌تواند برای منبع صبر کند و تعداد شروع مجددها کمتر می‌شود.

### شکستن شرط Circular Wait

این رایج‌ترین روش جلوگیری از بن‌بست است. اغلب سیستم‌ها با یک **قرارداد ساده** بین همه Threadها کار می‌کنند.

مثلاً در مثال Thread1 و Thread2 و منابع R1 و R2، اگر همه Threadها منابع را به همان ترتیب تخصیص دهند، انتظار دایره‌ای غیرممکن می‌شود.

اما مشکلاتی هم دارد:

* ترتیب گرفتن منابع ممکن است با ترتیب استفاده از آن‌ها هم‌خوانی نداشته باشد، بنابراین منابع ممکن است طولانی‌تر از نیاز قفل شوند.
* گاهی نمی‌توان ترتیب منابع را اعمال کرد، مثلاً وقتی ID منبع دوم از نتیجه منبع اول بدست می‌آید.

### نکته

راه‌های زیادی برای جلوگیری از بن‌بست وجود دارد، بعضی باعث **starvation** و بعضی مصرف بالای CPU و کاهش پاسخگویی می‌شوند.

ایزوله کردن بخش Threadها برای **تنظیم و آزمایش** راهی قدرتمند برای پیدا کردن بهترین استراتژی است.

---

### تست کد چندنخی

چگونه می‌توان تست نوشت تا نشان دهد کد زیر خراب است؟

```java
01: public class ClassWithThreadingProblem { 
02:    int nextId; 
03: 
04:    public int takeNextId() { 
05:        return nextId++; 
06:    } 
07:} 
```

شرح تست:

* مقدار فعلی `nextId` را یادداشت کنید.
* دو Thread ایجاد کنید که هرکدام `takeNextId()` را یک بار صدا بزنند.
* بررسی کنید که `nextId` دو واحد بیشتر از مقدار اولیه باشد.
* این کار را تکرار کنید تا ببینید `nextId` تنها یک واحد افزایش یافته است.

### نمونه تست (JUnit)

```java
01: package example;
02:
03: import static org.junit.Assert.fail;
04:
05: import org.junit.Test;
06:
07: public class ClassWithThreadingProblemTest {
08:     @Test
09:     public void twoThreadsShouldFailEventually() throws Exception {
10:         final ClassWithThreadingProblem classWithThreadingProblem
                = new ClassWithThreadingProblem();
11:
12:         Runnable runnable = new Runnable() {
13:             public void run() {
14:                 classWithThreadingProblem.takeNextId();
15:             }
16:         };
17:
18:         for (int i = 0; i < 50000; ++i) {
19:             int startingId = classWithThreadingProblem.lastId;
20:             int expectedResult = 2 + startingId;
21:
22:             Thread t1 = new Thread(runnable);
23:             Thread t2 = new Thread(runnable);
24:             t1.start();
25:             t2.start();
26:             t1.join();
27:             t2.join();
28:
29:             int endingId = classWithThreadingProblem.lastId;
30:
31:             if (endingId != expectedResult)
32:                 return;
33:         }
34:
35:         fail("Should have exposed a threading issue but it did not.");
36:     }
37: }
```

| Line  | Description                                                                                                                                                                                                                                               |
| ----- | --------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| 10    | ایجاد یک نمونه از کلاس **ClassWithThreadingProblem**. توجه داشته باشید که باید از کلیدواژه `final` استفاده کنیم زیرا در ادامه در یک کلاس داخلی ناشناس استفاده می‌شود.                                                                                     |
| 12–16 | ایجاد یک کلاس داخلی ناشناس که از همان نمونه استفاده می‌کند.                                                                                                                                                                                               |
| 18    | این کد را «به اندازه کافی» اجرا می‌کنیم تا نشان دهد کد خراب است، اما نه آن‌قدر زیاد که تست طولانی شود. این یک تعادل است؛ نمی‌خواهیم زمان زیادی منتظر شکست باشیم. انتخاب این عدد سخت است—اگرچه بعداً می‌بینیم می‌توانیم آن را به طور قابل توجهی کاهش دهیم. |
| 19    | مقدار شروع را به خاطر بسپارید. این تست قصد دارد ثابت کند کد در **ClassWithThreadingProblem** خراب است. اگر تست موفق شود، ثابت می‌کند کد خراب است. اگر تست شکست بخورد، تست نتوانست ثابت کند کد خراب است.                                                   |
| 20    | انتظار داریم مقدار نهایی دو واحد بیشتر از مقدار فعلی باشد.                                                                                                                                                                                                |
| 22–23 | ایجاد دو Thread که هر دو از نمونه ایجاد شده در خطوط 12–16 استفاده می‌کنند. این امکان را می‌دهد که دو Thread سعی کنند از همان نمونه استفاده کرده و با هم تداخل داشته باشند.                                                                                |



| Line  | Description                                                                                                                                                               |
| ----- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| 24–25 | اجازه می‌دهیم دو Thread ما آماده اجرا شوند.                                                                                                                               |
| 26–27 | منتظر می‌مانیم تا هر دو Thread قبل از بررسی نتایج به پایان برسند.                                                                                                         |
| 29    | مقدار نهایی واقعی را ثبت می‌کنیم.                                                                                                                                         |
| 31–32 | آیا مقدار `endingId` با چیزی که انتظار داشتیم متفاوت بود؟ اگر بله، تست را پایان می‌دهیم—ما ثابت کردیم کد خراب است. اگر نه، دوباره تلاش می‌کنیم.                           |
| 35    | اگر به اینجا رسیدیم، تست نتوانست ثابت کند کد تولیدی در «زمان معقول» خراب است؛ تست شکست خورده است. یا کد خراب نیست یا تعداد تکرارها برای رخ دادن شرایط خطا کافی نبوده است. |


این تست قطعاً شرایط لازم برای یک مشکل به‌روزرسانی هم‌زمان را فراهم می‌کند. با این حال، مشکل آن‌قدر به ندرت رخ می‌دهد که در اکثر مواقع این تست آن را تشخیص نمی‌دهد. در واقع، برای شناسایی واقعی مشکل، باید تعداد تکرارها را بیش از یک میلیون قرار دهیم. حتی در این صورت، در ده اجرای مختلف با شمارش حلقه ۱،۰۰۰،۰۰۰، مشکل تنها یک بار رخ داد. این یعنی احتمالاً باید تعداد تکرارها را به صد میلیون یا بیشتر افزایش دهیم تا شکست قابل اعتماد به دست آوریم. چقدر حاضر هستیم صبر کنیم؟

حتی اگر تست را طوری تنظیم کنیم که شکست قابل اعتماد روی یک ماشین حاصل شود، احتمالاً باید تست را با مقادیر متفاوت دوباره تنظیم کنیم تا شکست را روی ماشین، سیستم‌عامل یا نسخه دیگری از JVM نشان دهیم.

و این یک مشکل ساده است. اگر نتوانیم کد خراب را به راحتی با این مشکل نشان دهیم، چگونه می‌توانیم مشکلات پیچیده واقعی را تشخیص دهیم؟

پس چه رویکردهایی می‌توانیم برای نشان دادن این شکست ساده اتخاذ کنیم؟ و مهم‌تر از آن، چگونه می‌توانیم تست‌هایی بنویسیم که شکست‌ها را در کد پیچیده‌تر نشان دهند؟ چگونه می‌توانیم بفهمیم کد ما دارای خطا است وقتی نمی‌دانیم کجا به دنبال آن بگردیم؟

چند ایده وجود دارد:

* **تست مونت کارلو (Monte Carlo Testing):** تست‌ها را انعطاف‌پذیر طراحی کنید تا قابل تنظیم باشند. سپس تست را بارها اجرا کنید—مثلاً روی یک سرور تست—با تغییر تصادفی مقادیر تنظیم شده. اگر تست‌ها شکست خوردند، کد خراب است. اطمینان حاصل کنید که نوشتن این تست‌ها را زود شروع کنید تا یک سرور ادغام مداوم آنها را به زودی اجرا کند. همچنین، شرایطی که تست در آن شکست خورده را با دقت ثبت کنید.
* تست را روی **تمام پلتفرم‌های هدف** اجرا کنید. به‌طور مداوم. هرچه تست‌ها مدت طولانی‌تری بدون شکست اجرا شوند، احتمال بیشتری دارد که—کد تولیدی درست است یا—تست‌ها برای آشکار کردن مشکلات کافی نیستند.
* تست‌ها را روی ماشینی با **بارهای مختلف** اجرا کنید. اگر می‌توانید بارهایی نزدیک به محیط تولید شبیه‌سازی کنید، انجام دهید.

با این حال، حتی اگر همه این کارها را انجام دهید، هنوز شانس زیادی برای پیدا کردن مشکلات هم‌زمان در کد خود ندارید. پیچیده‌ترین مشکلات، آن‌هایی هستند که چنان نادرند که تنها یک بار در هر میلیارد فرصت رخ می‌دهند. این مشکلات، وحشت سیستم‌های پیچیده هستند.

### پشتیبانی ابزار برای تست کد مبتنی بر Thread

شرکت IBM ابزاری به نام **ConTest** ساخته است. این ابزار کلاس‌ها را ابزاردهی می‌کند تا احتمال شکست کد غیر هم‌زمان افزایش یابد.

ما هیچ رابطه مستقیمی با IBM یا تیم توسعه‌دهنده ConTest نداریم. یکی از همکاران ما به آن اشاره کرد و ما پس از چند دقیقه استفاده، بهبود قابل توجهی در توانایی یافتن مشکلات هم‌زمان مشاهده کردیم.

راهنمای استفاده از ConTest:

* تست‌ها و کد تولیدی را بنویسید، اطمینان حاصل کنید که تست‌ها به‌طور خاص برای شبیه‌سازی کاربران متعدد با بارهای متغیر طراحی شده‌اند.
* تست و کد تولیدی را با ConTest ابزاردهی کنید.
* تست‌ها را اجرا کنید.

وقتی کد را با ConTest ابزاردهی کردیم، نرخ موفقیت ما از تقریباً یک شکست در ده میلیون تکرار به تقریباً یک شکست در سی تکرار رسید. مقادیر حلقه برای چند اجرای تست پس از ابزاردهی: ۱۳، ۲۳، ۰، ۵۴، ۱۶، ۱۴، ۶، ۶۹، ۱۰۷، ۴۹، ۲. بنابراین واضح است که کلاس‌های ابزاردهی شده خیلی زودتر و با قابلیت اطمینان بیشتری شکست خوردند.

### نتیجه‌گیری

این فصل یک سفر بسیار کوتاه در قلمرو بزرگ و خطرناک برنامه‌نویسی هم‌زمان بود. ما فقط سطح آن را لمس کردیم. تمرکز ما بر روی اصولی بود که کمک می‌کند کد هم‌زمان پاک بماند، اما چیزهای بیشتری وجود دارد که باید یاد بگیرید اگر می‌خواهید سیستم‌های هم‌زمان بنویسید. ما توصیه می‌کنیم با کتاب فوق‌العاده **Doug Lea با عنوان Concurrent Programming in Java: Design Principles and Patterns** شروع کنید.

در این فصل درباره به‌روزرسانی هم‌زمان، اصول همگام‌سازی و قفل‌گذاری تمیز که می‌تواند از آن جلوگیری کند صحبت کردیم. درباره اینکه چگونه Threadها می‌توانند توان عملیاتی سیستم‌های I/O محور را افزایش دهند و تکنیک‌های تمیز برای دستیابی به این بهبودها صحبت کردیم. درباره بن‌بست و اصول پیشگیری از آن به روش تمیز بحث کردیم. در نهایت، درباره استراتژی‌های آشکار کردن مشکلات هم‌زمان با ابزاردهی کد صحبت کردیم.


---

### آموزش: مثال‌های کامل کد

#### کلاینت/سرور غیرهم‌زمان (Nonthreaded)

**لیست A-3 – Server.java**

```java
package com.objectmentor.clientserver.nonthreaded;

import java.io.IOException;
import java.net.ServerSocket;
import java.net.Socket;
import java.net.SocketException;
import common.MessageUtils;

public class Server implements Runnable {
    ServerSocket serverSocket;
    volatile boolean keepProcessing = true;

    public Server(int port, int millisecondsTimeout) throws IOException {
        serverSocket = new ServerSocket(port);
        serverSocket.setSoTimeout(millisecondsTimeout);
    }

    public void run() {
        System.out.printf("Server Starting\n");
        while (keepProcessing) {
            try {
                System.out.printf("accepting client\n");
                Socket socket = serverSocket.accept();
                System.out.printf("got client\n");
                process(socket);
            } catch (Exception e) {
                handle(e);
            }
        }
    }

    private void handle(Exception e) {
        if (!(e instanceof SocketException)) {
            e.printStackTrace();
        }
    }

    public void stopProcessing() {
        keepProcessing = false;
        closeIgnoringException(serverSocket);
    }

    void process(Socket socket) {
        if (socket == null)
            return;
        try {
            System.out.printf("Server: getting message\n");
            String message = MessageUtils.getMessage(socket);
            System.out.printf("Server: got message: %s\n", message);
            Thread.sleep(1000);
            System.out.printf("Server: sending reply: %s\n", message);
            MessageUtils.sendMessage(socket, "Processed: " + message);
            System.out.printf("Server: sent\n");
            closeIgnoringException(socket);
        } catch (Exception e) {
            e.printStackTrace();
        }
    }

    private void closeIgnoringException(Socket socket) {
        if (socket != null)
            try {
                socket.close();
            } catch (IOException ignore) {
            }
    }

    private void closeIgnoringException(ServerSocket serverSocket) {
        if (serverSocket != null)
            try {
                serverSocket.close();
            } catch (IOException ignore) {
            }
    }
}
```

---

**لیست A-4 – ClientTest.java**
(توضیح: کد ClientTest عملاً مشابه Server.java است و همان ساختار را دارد.)

```java
package com.objectmentor.clientserver.nonthreaded;

import java.io.IOException;
import java.net.ServerSocket;
import java.net.Socket;
import java.net.SocketException;
import common.MessageUtils;

public class Server implements Runnable {
    ServerSocket serverSocket;
    volatile boolean keepProcessing = true;

    public Server(int port, int millisecondsTimeout) throws IOException {
        serverSocket = new ServerSocket(port);
        serverSocket.setSoTimeout(millisecondsTimeout);
    }

    public void run() {
        System.out.printf("Server Starting\n");
        while (keepProcessing) {
            try {
                System.out.printf("accepting client\n");
                Socket socket = serverSocket.accept();
                System.out.printf("got client\n");
                process(socket);
            } catch (Exception e) {
                handle(e);
            }
        }
    }

    private void handle(Exception e) {
        if (!(e instanceof SocketException)) {
            e.printStackTrace();
        }
    }

    public void stopProcessing() {
        keepProcessing = false;
        closeIgnoringException(serverSocket);
    }

    void process(Socket socket) {
        if (socket == null)
            return;
        try {
            System.out.printf("Server: getting message\n");
            String message = MessageUtils.getMessage(socket);
            System.out.printf("Server: got message: %s\n", message);
            Thread.sleep(1000);
            System.out.printf("Server: sending reply: %s\n", message);
            MessageUtils.sendMessage(socket, "Processed: " + message);
            System.out.printf("Server: sent\n");
            closeIgnoringException(socket);
        } catch (Exception e) {
            e.printStackTrace();
        }
    }

    private void closeIgnoringException(Socket socket) {
        if (socket != null)
            try {
                socket.close();
            } catch (IOException ignore) {
            }
    }

    private void closeIgnoringException(ServerSocket serverSocket) {
        if (serverSocket != null)
            try {
                serverSocket.close();
            } catch (IOException ignore) {
            }
    }
}
```

---

**لیست A-5 – MessageUtils.java**

```java
package common;

import java.io.IOException;
import java.io.InputStream;
import java.io.ObjectInputStream;
import java.io.ObjectOutputStream;
import java.io.OutputStream;
import java.net.Socket;

public class MessageUtils {
    public static void sendMessage(Socket socket, String message) throws IOException {
        OutputStream stream = socket.getOutputStream();
        ObjectOutputStream oos = new ObjectOutputStream(stream);
        oos.writeUTF(message);
        oos.flush();
    }

    public static String getMessage(Socket socket) throws IOException {
        InputStream stream = socket.getInputStream();
        ObjectInputStream ois = new ObjectInputStream(stream);
        return ois.readUTF();
    }
}
```

---

### کلاینت/سرور با استفاده از Threads

تغییر سرور برای استفاده از threads فقط نیازمند تغییر در متد **process** پیام است (خطوط جدید برجسته شده‌اند):

```java
void process(final Socket socket) {
    if (socket == null)
        return;

    Runnable clientHandler = new Runnable() {
        public void run() {
            try {
                System.out.printf("Server: getting message\n");
                String message = MessageUtils.getMessage(socket);
                System.out.printf("Server: got message: %s\n", message);
                Thread.sleep(1000);
                System.out.printf("Server: sending reply: %s\n", message);
                MessageUtils.sendMessage(socket, "Processed: " + message);
                System.out.printf("Server: sent\n");
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

در این نسخه، هر اتصال کلاینت در یک **Thread جداگانه** اجرا می‌شود تا سرور بتواند همزمان با چندین کلاینت کار کند.
