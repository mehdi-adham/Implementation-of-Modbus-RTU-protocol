# پیاده سازی پروتکل مدباس آرتیو

## پیکربندی ارتباط سریال و آدرس دستگاه

1. انتخاب حالت ASCII یا RTU (که فقط به شبکه های استاندارد Modbus مربوط می شود)

```C

typedef enum Serial_Transmission_Modes { 
	RTU,
	ASCII
} Serial_Transmission_Modes_t;
```

2. کاربران مد مورد نظر را به همراه پارامترهای ارتباطی پورت سریال (نرخ baud، مد parity و غیره) در طول پیکربندی هر کنترلر انتخاب می کنند. پارامترهای مد و سریال باید برای همه دستگاه های موجود در یک شبکه Modbus یکسان باشد.


```C

typedef enum Parity{
	EVEN,
	ODD,
	NONE
} Parity_t;

typedef enum Stop_Bit{
	StopBit_1,
	StopBit_2
} Stop_Bit_t;

typedef struct Serial {
	int UART;
	int BaudRate;
	Parity_t Parity;
	Stop_Bit_t StopBit;
} Serial_t;
```


در زمان روشن شدن یا زمان راه اندازی مجدد، دستگاه کانفیگ می شود.

موارد زیر باید تنظیم شوند:
1.	نوع اتصال (سریال – TCP/IP)   فعلا فقط سریال هست
2.	تنظیم ارتباط سریال
3.	تنظیم آدرس دستگاه


بعد از تنطیم موارد بالا تابع modbus_init فراخوانی شود جهت اعمال تنظیمات
برای تنظیم سریال یک تابع هندلر  از نوع weak تعریف می کنم و بعد آن را در برنامه خودم پیاده سازی می کنم. 


```C

__attribute__((weak))	 void modbus_uart_init_Handler(Serial_t *Serial) {

}
```


**مثلا STM32  با توابع HAL می شود:**

```C

void modbus_uart_init_Handler(Serial_t *Serial) {
	huart1.Instance = USART1;
	huart1.Init.BaudRate = Serial->BaudRate;

	if(Serial->StopBit == StopBit_1)
		huart1.Init.StopBits = UART_STOPBITS_1;
	else
		huart1.Init.StopBits = UART_STOPBITS_2;

	if(Serial->Parity == NONE_PARITY)
		huart1.Init.Parity = UART_PARITY_NONE;
	else if(Serial->Parity == ODD_PARITY)
		huart1.Init.Parity = UART_PARITY_ODD;
	else
		huart1.Init.Parity = UART_PARITY_EVEN;

	huart1.Init.WordLength = UART_WORDLENGTH_8B;
	huart1.Init.Mode = UART_MODE_TX_RX;
	huart1.Init.HwFlowCtl = UART_HWCONTROL_NONE;
	huart1.Init.OverSampling = UART_OVERSAMPLING_16;

	HAL_UART_Init(&huart1);
}
```

## نحوه استفاده از توابع در برنامه به صورت عادی:

```C

set_slave_ID(17);

	Serial_t default_serial;
	default_serial.BaudRate = 9600;
	default_serial.Parity = NONE_PARITY;
	default_serial.StopBit = StopBit_1;
	modbus_serial_init(&default_serial);
```


## تنظیم دستگاه با حالتهای مختلف (دیپ سوئیچ)

- اگر هر 8 سوئیج OFF بودند وارد مد تنظیمات می شود.( برای تنظیم آدرس جدید یا تغییر پارامترهای سریال به کمک نرم افزار روی RS-485)
- اگر هر 4 سوئیچ ON بودند تنظیمات نرم افزاری را اعمال می کند. 
- اگر غیر از موارد بالا بود تنظیمات سخت افزاری اعمال می شود آدرس دستگاه و باودریت با توجه دیپ سوئیچ تنظیم خواهد شد.
ابتدا قبل از همه پین های دیپ سویچ را پیکربندی می کنیم:
بعد شروط  را قرار می دهیم: این با میکرو STM32 و کتابخانه HAL 


```C

	/* Get address from DIP switch */
	uint8_t dip_switch_address = (!HAL_GPIO_ReadPin(DS_1_GPIO_Port, DS_1_Pin))
			| (!HAL_GPIO_ReadPin(DS_2_GPIO_Port, DS_2_Pin) << 1)
			| (!HAL_GPIO_ReadPin(DS_3_GPIO_Port, DS_3_Pin) << 2)
			| (!HAL_GPIO_ReadPin(DS_4_GPIO_Port, DS_4_Pin) << 3)
			| (!HAL_GPIO_ReadPin(DS_5_GPIO_Port, DS_5_Pin) << 4);

	/* Get Baud rate from DIP switch */
	uint8_t dip_switch_baudrate = (!HAL_GPIO_ReadPin(DS_6_GPIO_Port, DS_6_Pin))
			| (!HAL_GPIO_ReadPin(DS_7_GPIO_Port, DS_7_Pin) << 1)
			| (!HAL_GPIO_ReadPin(DS_8_GPIO_Port, DS_8_Pin) << 2);



	Serial_t serial;
	uint32_t baudrate;

	/* Soft Setting...  */
	if(dip_switch_address == 0){
		Soft_Setting();
	}
	/* GET Parameter from Soft Setting (Flash)...  */
	else if(dip_switch_address == 31){
		uint8_t soft_setting_data[4];

		FLASH_Read_for_Soft_Setting_Data((uint32_t *)soft_setting_data);

		set_slave_ID(soft_setting_data[0]);
		baudrate = Select_BaudRate(soft_setting_data[1]);
		Select_Format(soft_setting_data[2], &serial);

	}
	/* GET Parameter from Hard Setting (dip switch)... */
	else{
		set_slave_ID(dip_switch_address);
		baudrate = Select_BaudRate(dip_switch_baudrate);

		serial.DataLength = DataLength_8;
		serial.Parity = NONE_PARITY;
		serial.StopBit = StopBit_1;
	}

	serial.UART = (uint32_t*) USART1;
	serial.BaudRate = baudrate;
	modbus_serial_init(&serial);

```

## اسکن شبکه مدباس

1.	دستگاه های تحت شبکه به طور مداوم باس شبکه را مانیتور می کنند، از جمله در فواصل سکوت یا silent. در حالت RTU، پیام‌ها با فاصله سکوت یا silent حداقل زمان 3.5 کاراکتر شروع می‌شوند.
2.	هنگامی که اولین فیلد (فیلد آدرس) دریافت می شود، هر دستگاه آن را رمزگشایی می کند تا بفهمد آیا برای آن دستگاه آدرس دهی شده است یا خیر.
3.	کل فریم پیام باید به صورت یک جریان پیوسته منتقل شود. اگر قبل از اتمام فریم فاصله سکوت بیش از 1.5 کاراکتر بار رخ دهد، دستگاه دریافت کننده پیام ناقص را پاک می کند و فرض می کند که بایت بعدی فیلد آدرس یک پیام جدید است.



**مستر** محتوای فریم پاسخ از اسلیو را بررسی می کند که:
1. شناسه آدرس اسلیو همانی هست که من درخواست داده بودم 
2. فانکشن کد همانی است که من (مستر) دخواست داده بودم اگر نه یعنی اسلیو یک پیام خطا فرستاده
3. بررسی فیلد خطا جهت اینکه آیا اطلاعات فریم معتبر است یا نه


**اسلیو** به طور مداوم شبکه را نظارت می کنداگر شبکه به فاصله حداقل 3.5 کاراکتر سکوت یا silent باشد کنترلر می تواند آماده باشد که اولین فیلد (فیلد آدرس) را دریافت کند.

این به این معنی است که هر بار اسلیو ما شبکه را مانیتور می کند، باید سطح منطقی پین RX را بررسی کند که آیا فاصله حداقل 3.5 کاراکتر یا بیشتر صفر است یا نه؟ اگر نبود یعنی مستر یا یک اسلیو دیگر در حال ارسال یک فریم هستند. پس اسلیو باید صبر کند.

 اگر در برنامه برای مانیتور کردن شبکه تایم اوت داریم یعنی مثلا اگر بیش تر از 500 میلی ثانیه یا 1 ثانیه هیچ ریکوئستی  وجود نداشت یا برای دستگاه ما وجود نداشت، از تابع مانیتور خارج شود چون به نظر می رسد که روی شبکه هیچ ریکوئستی از سمت مستر ارسال نمی شود (یا حداقل برای دستگاه ما) و پردازنده در تابع مانیتور بیش از اندازه درگیر نباشد و به تابع اصلی نیز برگردد. 

**نکته**: طول زمانی یک فریم نمی تواند بیشتر از 500ms باشد مثلا با پارامترهای زیر طول یک فریم با 250 بایت که خیلی زیاد هست می شود 208 میلی ثانیه:

```text

max frame lenth = 250 byte
baud rate = 9600 (9600 Bit/Sec)
250 *  8 * (1/9600) = 208ms
```

بعد از دریافت اولین فیلد (آدرس) دقت کنید که اگر قبل از اتمام فریم فاصله سکوت بیش از 1.5 کاراکتر شود اسلیو باید پیام ناقص را پاک کند و فرض کند که بایت بعدی فیلد آدرس یک پیام جدید است. در واقع فاصله بین بایت های دریافتی باید کمتر از زمان 1.5 کاراکتر باشد.

به طور مشابه در سمت اسلیو نیز 3.5 و 1.5 کاراکتر را برای پاسخ به مستر باید رعایت کند. یعنی بعد از دریافت یک فریم باید برای ارسال پاسخ به مدت 3.5 کاراکتر فاصله ایجاد کند و همچنین بایت های یک فریم ارسالی نباید بیشتر 1.5 کاراکتر بین آنها فاصله باشد و باید به صورت پیوسته ارسال شود.

## نحوه پیاده سازی تابع مانیتور شبکه مدباس MODBUS_RTU_MONITOR


در تابع اصلی برنامه برای گوش دادن به شبکه مدباس که آیا درخواستی از سمت مستر برای دیوایس ما ارسال شده است یا نه، این تابع را فراخوانی می کنیم. این تابع دارای مهلت زمانی (تایم اوت) است تا اگر در مهلت زمانی، ریکوئستی روی شبکه دریافت نکرد و یا اصلا برای دیوایس ما ریکوئستی روی شبکه نبود از تابع خارج شود و به تابع اصلی برگردد.
به معنای دیگر در آن مهلت زمانی صبر می کند تا حداقل یکبار مستر برای دیواس ما درخواستی ارسال کند.
این به دیوایس ما کمک می کند در این فاصله برنامه اصلی را نیز اجرا کند. و بعد از آن باز با تابع MODBUS_RTU_MONITOR
 شبکه را بررسی کند.


- مهلت زمانی برای دریافت ریکوئستی مربوط به این دیوایس Timeout to receive request this device
- مهلت زمانی برای دریافت از ابتدای فریم.5C 3  Timeout to receive frame
- مهلت زمانی برای دریافت بایت های یک فریم از یوارت1.5C  Timeout for receiving bytes


**مهلت زمانی اول:** از زمانی که تابع MODBUS_RTU_MONITOR فراخوانی می شود شروع به شمردن می کند و تا زمانی که ریکوئستی با آدرس این دیوایس دریافت نکند به شمردن ادامه می دهد مثلا اگر 500 میلی ثانیه بود در این مدت اگر ریکوئستی با آدرس دیوایس ما دریافت نکرد از تابع خارج می شود و بیشتر از این شبکه را مانیتور نمی کند تا دوباره فراخوانی شود.
نکته: ممکن است در زمان مانیتور شبکه، ریکوئست هایی دریافت کند اما برای این دیوایس نباشد.
**مهلت زمانی دوم:** برای دریافت صحیح از ابتدای فریم، شبکه مدباس باید حداقل به اندازه زمان ارسال 3.5 کاراکتر در سکوت باشد( یعنی پین مربوط به دریافت RX در منطق صفر باشد) بعد از بایت اول و بایت های بعدی تا به انتها پشت سر هم به صورت پیوسته دریافت شود.
**مهلت زمانی سوم:** بین بایت ها دریافتی نباید بیشتر از زمان ارسال 1.5 کاراکتر فاصله باشد.


## نحوه پیاده سازی مهلت زمانی:

در کد زیر تایمر ریکوئست را با مقدار مهلت زمانی ریکوئست، مقدار دهی می کنیم. تابع تایمر هر 1 میلی ثانیه، اجرا میشود و تایمر ریکوئست رو کم می کند تا به صفر برسد. در برنامه مقدار تایمر ریکوئست را بررسی می کنیم، اگر صفر بود یعنی مهلت زمانی به پایان رسیده است.

```C

/* Start request timer for this device*/
request_timer = request_timeout;
```

```C

void Timer()
{
    if (request_timer)
    {
        request_timer--;
    }
}
```

البته می توان به شکل دیگری نیز مهلت زمانی را پیاده سازی کرد و آن این است که مقدار تایمر سیستم یا SysTick را در یک متغیر ذخیره می کنیم و با فاصله دوباره مقدار تایمر سیستم را در یک متغیر دیگر ذخیره می کنیم سپس آن دو را از هم کم می کنیم اگر نتیجه بزرگتر از مهلت زمانی بود یعنی مهلت زمانی به پایان رسیده.  
نکته: دقت کنید که تایمر سسیستم به میلی ثانیه است یا میکرو ثانیه و آن را در محاسبات خود در نظر بگیرید.

```C

	uint32_t tickstart_for_monitor_timeout;
	uint32_t currenttick_for_monitor_timeout;

	/* Initial tick start for monitor timeout management */
	tickstart_for_monitor_timeout = *Tick;
	/* 0. Check for monitor function timeout */
		currenttick_for_monitor_timeout = *Tick;
		if (currenttick_for_monitor_timeout - tickstart_for_monitor_timeout > monitor_fun_timeout)
			return MODBUS_MONITOR_TIMEOUT;
```

## مراحل پیاده سازی تابع مانیتور شبکه مدباس:  
0. بررسی مهلت زمانی مانیتور شبکه مدباس اگر در این فاصله ریکوئیسی به آدرس دیوایس ما ارسال نشده بود از تابع خارج شود.
1.	بررسی مهلت زمانی فریم، برای دریافت فریم جدید. منتظر بماند تا مهلت زمانی فریم رخ دهد.
2.	بعد از اینکه مهلت زمانی فریم رخ داد، آماده شود برای دریافت بایت ¬نخست از فریم¬جدید، پس تابع دریافت یوارت فراخوانی شود.
3.	بعد از برگشت از تابع دریافت یوارت اگر مهلت زمانی رخ نداده بود، آنوقت بررسی می کنیم که اگر مقدار بایت در محدوده آدرس مجاز نبود و یا با آدرس دستگاه ما یکی نبود به شماره 0 برگردد.
4.	در صورتی که بایت دریافتی با آدرس دستگاه ما یکی بود مقدار آن در پوینتر buffer ذخیره شود و crc آن محاسبه شود و آماده شود برای دریافت بایت های بعدی فریم(پس این فریم مربوط به دستگاه ماست).
5.	تابع دریافت یوارت فراخوانی شود. 
6.	بعد از برگشت از تابع دریافت یوارت اگر مهلت زمانی رخ نداده بود مقدار آن در متغیر fun ذخیره شود (این بایت دوم فانکشن¬کد می باشد) و همچنین در پوینتر buffer ذخیره شود و crc آن محاسبه شود و آماده شود برای دریافت بایت های بعدی فریم.
7.	محاسبه تعداد بایت های بعدی با توجه به مقدار fun  یا فانکشن¬کد و ذخیره آن در متغیر len


**استخراج تعداد بایت های بعدی:**

محاسبه تعداد بایت های بعدی با توجه به فانکشن های مدباس تا قبل از دو بایت CRC (بدون احتساب دو بایت CRC).در این روش می توان همزمان crc داده ها را محاسبه کرد و در انتها با crc فریم مقایسه کرد در صورت نابرابری مقدار false بر می گردد و مقادیر پوینتر بافر پاک می شود. 
**نکته**:هربار که بایتی از یوارت دریافت می شود در بافر ذخیره شود و crc محاسبه شود


| ردیف | دستورات | توضیحات |
| --- | --- | --- |
| 1 | 01 Read Coil Status  <br> 02 Read Input Status <br>03 Read Holding Registers <br> 04 <br>Read Input Registers <br>05 Force Single Coil<br>06 Preset Single Register | <p dir="rtl">اینجا بعد از فانکشن مشخصا 4 بایت دیگر دریافت می کند بدون احتساب دو بایت CRC </p>|
| 2 | 07 Read Exception Status 11 (0B Hex) Fetch Comm Event Counter<br>12 (0C Hex) Fetch Comm Event Log<br>17 (11 Hex) Report Slave ID | <p dir="rtl"> اینجا بعد از فانکشن CRC می آید. </p> |
| 3 | 15 (0F Hex) Force Multiple Coils <br> 16 (10 Hex) Preset Multiple Registers | <p dir="rtl"> بایت هفتم تعداد بایت های بعد از خودش را مشخص می کند بدون احتساب دو بایت CRC </p>|
| 4 | 20 (14Hex) Read General Reference <br> 21 (15Hex) Write General Reference | <p dir="rtl"> بایت سوم تعداد بایت های بعد از خودش را مشخص می کند بدون احتساب دو بایت CRC </p> |
| 5 | 22 (16Hex) Mask Write 4X Register | <p dir="rtl"> اینجا بعد از فانکشن مشخصا 6 بایت دیگر دریافت می کند بدون احتساب دو بایت CRC </p>  |
| 6 | 23 (17Hex) Read/Write 4X Registers | <p dir="rtl"> بایت یازدهم تعداد بایت های بعد از خودش را مشخص می کند بدون احتساب دو بایت CRC  </p>  |
| 7 | 24 (18Hex) Read FIFO Queue |  <p dir="rtl">  اینجا بعد از فانکشن مشخصا 2 بایت دیگر دریافت می کند بدون احتساب دو بایت CRC  </p> |


8.	بعد از محاسبه طول بایت های بعدی(بدون احتساب دو بایت crc)، بایت های بعدی را داخل حلقه وایل دریافت می کنیم. عملیات داخل حلقه:
8. 1.	تابع دریافت یوارت فراخوانی شود.
8. 2.	بعد از برگشت از تابع دریافت یوارت اگر مهلت زمانی رخ نداده بود مقدار آن در پوینتر buffer ذخیره شود و crc آن محاسبه شود.
8. 3.	متغیرlen  یک واحد کم شود.
9.	بعد از خروج از حلقه وایل دو بایت آخر یعنی crc دریافت شود بدون محاسبه crc آنها.
10.	crc محاسبه شده با crc دریافت شده از فریم مقایسه شود اگر یکی نبود متغیر buffer پاک شود و به شماره0 برگردد(ابتدای تابع)
11.	اگر crc یکی بود پس اطلاعات بدرستی در بافر buffer ذخیره شده است و تابع پردازش فریم MODBUS_FARME_PROCESS فراخوانی شود.


```C


/**
 * @brief This function receives the frame related to this device from serial, 
 * processes it and applies commands (reading/writing to registers or coils) and 
 * prepares the response or exception response for the master and sends it via serial.
 * 
 * @note Template Frame For Test:
 * {0x11, 0x01, 0x00, 0x13, 0x00, 0x25, 0x0E, 0x84}
 * {0x11, 0x02, 0x00, 0xC4, 0x00, 0x16, 0xBA, 0xA9}
 * {0x11, 0x03, 0x00, 0x6B, 0x00, 0x03 ,0x76 ,0x87}
 * {0x11, 0x04, 0x00, 0x08, 0x00, 0x01 ,0xB2 ,0x98}
 * {0x11, 0x05, 0x00, 0xAC, 0xFF, 0x00 ,0x4E ,0x8B}
 * {0x11, 0x06, 0x00, 0x01, 0x00, 0x03 ,0x9A ,0x9B}
 * {0x11, 0x07, 0x4C, 0x22}
 * {0x11, 0x0B, 0x4C, 0x27}
 * {0x11, 0x0C, 0x0D, 0xE5}
 * {0x11, 0x0F, 0x00, 0x13, 0x00, 0x0A ,0x02 ,0xCD, 0x01, 0xBF, 0x0B}
 * {0x11, 0x10, 0x00, 0x01, 0x00, 0x02 ,0x04 ,0x00, 0x0A, 0x01, 0x02, 0xC6, 0xF0}
 * {0x11, 0x11, 0xCD, 0xEC}
 * {0x11, 0x14, 0x0E, 0x06, 0x00, 0x04 ,0x00 ,0x01, 0x00, 0x02, 0x06, 0x00, 0x03, 0x00, 0x09, 0x00, 0x02, 0xF9, 0x38}
 * {0x11, 0x15, 0x0D, 0x06, 0x00, 0x04 ,0x00 ,0x07, 0x00, 0x03, 0x06, 0xAF, 0x04, 0xBE, 0x10, 0x0D, 0xDB, 0xC7}
 * {0x11, 0x16, 0x00, 0x04, 0x00, 0xF2 ,0x00 ,0x25, 0x66, 0xE2}
 * {0x11, 0x17, 0x00, 0x04, 0x00, 0x06 ,0x00 ,0x0F, 0x00, 0x03, 0x06, 0x00, 0xFF, 0x00, 0xFF, 0x00, 0xFF, 0x1C, 0x56}
 * {0x11, 0x18, 0x04, 0xDE, 0x07, 0x87}
 *
 * @param mbus_frame_buffer Saves the frame in the user buffer
 * @param Tick Get a pointer to tick value in millisecond. For 
 * setting the timeout to exit the function if the master device 
 * does not send the frame related to this device
 * @param Mode 1. Normal mode (by responding to the master and applying commands) 
 * 2. only listening mode (without responding to the master and without applying commands)
 */
ModbusStatus_t MODBUS_RTU_MONITOR(unsigned char *mbus_frame_buffer,
		int monitor_fun_timeout, volatile uint32_t *Tick, ModbusMonitorMode_t Mode) {

	unsigned char uchCRCHi = 0xFF; /* high byte of CRC initialized */
	unsigned char uchCRCLo = 0xFF; /* low byte of CRC initialized */
	unsigned char uIndex; /* will index into CRC lookup table */
	unsigned short calculate_crc;
	unsigned char rec_byte;
	ModbusStatus_t res;

	uint32_t tickstart_for_monitor_timeout;
	uint32_t currenttick_for_monitor_timeout;

	/* The starting address of the buffer */
	unsigned char *starting_address_of_buffer = mbus_frame_buffer;

	unsigned char (*receive_uart_fun)() = modbus_uart_receive_Handler;
	void (*transmit_uart_fun)(uint8_t *Data,
			uint16_t length) = modbus_uart_transmit_Handler;
	uint16_t counter = 0;

	unsigned char SLAVE_ADDRESS = get_slave_ID();

	/* Initial tick start for monitor timeout management */
	tickstart_for_monitor_timeout = *Tick;
	while (1) {
		/* 0. Check for monitor function timeout */
		currenttick_for_monitor_timeout = *Tick;
		if (currenttick_for_monitor_timeout - tickstart_for_monitor_timeout
				> monitor_fun_timeout)
			return MODBUS_MONITOR_TIMEOUT;

		/* 1. frame timeout management */
		do {
			res = (*receive_uart_fun)(&rec_byte);
		} while (res == MODBUS_OK);

		/* 2. Ready for receive of first Byte (Address Field) */
		do {
			res = (*receive_uart_fun)(&rec_byte);
			/* Check for monitor function timeout */
			currenttick_for_monitor_timeout = *Tick;
			if (currenttick_for_monitor_timeout - tickstart_for_monitor_timeout
					> monitor_fun_timeout)
				return MODBUS_MONITOR_TIMEOUT;
		} while (res != MODBUS_OK);


		/* 3. if out of range allowed address OR Address field not match with slave ID AND not broadcast */
		if (rec_byte > MAX_SLAVE_ADDRESS || (rec_byte != SLAVE_ADDRESS && rec_byte != Broadcast))
			/* return to 0. */
			continue;

		uchCRCHi = 0xFF;
		uchCRCLo = 0xFF;

		/* 4. Be assigned to the buffer and Calculate the CRC of this field */
		frame_buffer[counter++] = *mbus_frame_buffer++ = rec_byte;
		uIndex = uchCRCHi ^ rec_byte;
		uchCRCHi = uchCRCLo ^ auchCRCHi[uIndex];
		uchCRCLo = auchCRCLo[uIndex];

		/* 5. Get second field (Function Field) */
		res = (*receive_uart_fun)(&rec_byte);
		if (res != MODBUS_OK)
			return res;

		/* 6. Be assigned to the fun/buffer value and Calculate the CRC of this field */
		unsigned char fun = frame_buffer[counter++] = *mbus_frame_buffer++ =
				rec_byte;
		uIndex = uchCRCHi ^ rec_byte;
		uchCRCHi = uchCRCLo ^ auchCRCHi[uIndex];
		uchCRCLo = auchCRCLo[uIndex];


		/* 7. */
		/* Extracting length (register or coil ...) from frame data */
		int len = 0;
		if (fun == Read_Coil_Status || fun == Read_Input_Status
				|| fun == Read_Holding_Registers || fun == Read_Input_Registers
				|| fun == Force_Single_Coil || fun == Preset_Single_Register) {
			len = 4;
		} else if (fun == Read_Exception_Status
				|| fun == Fetch_Comm_Event_Counter
				|| fun == Fetch_Comm_Event_Log || fun == Report_Slave_ID) {
			len = 0;
		} else if (fun == Force_Multiple_Coils
				|| fun == Preset_Multiple_Registers) {
			int l = 5;
			while (l--) {
				res = (*receive_uart_fun)(&rec_byte);
				if (res != MODBUS_OK)
					return res;
				frame_buffer[counter++] = *mbus_frame_buffer++ = rec_byte;
				/* Calculate the CRC of this field */
				uIndex = uchCRCHi ^ rec_byte;
				uchCRCHi = uchCRCLo ^ auchCRCHi[uIndex];
				uchCRCLo = auchCRCLo[uIndex];
			}

			len = rec_byte;
		} else if (fun == Read_General_Reference
				|| fun == Write_General_Reference) {
			res = (*receive_uart_fun)(&rec_byte);
			if (res != MODBUS_OK)
				return res;
			frame_buffer[counter++] = *mbus_frame_buffer++ = rec_byte;
			/* Calculate the CRC of this field */
			uIndex = uchCRCHi ^ rec_byte;
			uchCRCHi = uchCRCLo ^ auchCRCHi[uIndex];
			uchCRCLo = auchCRCLo[uIndex];

			len = rec_byte;
		} else if (fun == Mask_Write_4X_Register) {
			len = 6;
		} else if (fun == Read_Write_4X_Registers) {
			int l = 9;
			while (l--) {
				res = (*receive_uart_fun)(&rec_byte);
				if (res != MODBUS_OK)
					return res;
				frame_buffer[counter++] = *mbus_frame_buffer++ = rec_byte;
				/* Calculate the CRC of this field */
				uIndex = uchCRCHi ^ rec_byte;
				uchCRCHi = uchCRCLo ^ auchCRCHi[uIndex];
				uchCRCLo = auchCRCLo[uIndex];
			}

			len = rec_byte;
		} else if (fun == Read_FIFO_Queue) {
			len = 2;
		}


		/* 8. Remain byte */
		while (len) {
			/* check overflow */
			if(counter > MAX_BUFFER){
				/* clear buffer */
				while (starting_address_of_buffer < mbus_frame_buffer) {
					frame_buffer[counter--] = *mbus_frame_buffer-- = 0x00;
				}
				/* return to 0. */
				continue;
			}

			/* 8.1 */
			res = (*receive_uart_fun)(&rec_byte);
			if (res != MODBUS_OK)
				return res;
			/* 8.2 */
			frame_buffer[counter++] = *mbus_frame_buffer++ = rec_byte;
			/* Calculate the CRC of this field */
			uIndex = uchCRCHi ^ rec_byte;
			uchCRCHi = uchCRCLo ^ auchCRCHi[uIndex];
			uchCRCLo = auchCRCLo[uIndex];

			/* 8.3 */
			len--;
		}

		/* 9. Get CRC */
		res = (*receive_uart_fun)(&rec_byte);
		if (res != MODBUS_OK)
			return res;
		frame_buffer[counter++] = *mbus_frame_buffer++ = rec_byte;

		res = (*receive_uart_fun)(&rec_byte);
		if (res != MODBUS_OK)
			return res;
		frame_buffer[counter++] = *mbus_frame_buffer++ = rec_byte;

		/* 10. Check CRC calculated, that is equal with CRC frame */
		calculate_crc = (uchCRCHi << 8 | uchCRCLo);
		unsigned short frameCRC = (unsigned short) ((*(mbus_frame_buffer - 2)
				<< 8) | *(mbus_frame_buffer - 1));

		if (calculate_crc != frameCRC) {
			/* clear buffer */
			while (starting_address_of_buffer < mbus_frame_buffer) {
				frame_buffer[counter--] = *mbus_frame_buffer-- = 0x00;
			}
			/* return to 0. */
			continue;
		}

		break;
	}/*< End while() for frame time out */


	if(Mode == Normal){

		/* 10. MODBUS PROCESS for Constructed Response Frame */
		uint8_t len = MODBUS_FARME_PROCESS(frame_buffer, response_buffer);


		/* Add CRC to response frame */
	    unsigned short crc = CRC16(response_buffer, len);
		response_buffer[len] = crc >> 8; /* CRC Lo */
		response_buffer[len + 1] = crc; /* CRC Hi */

		len += 2;

		/* 11. Transmit frame */
		(*transmit_uart_fun)(response_buffer, len);

	}

	return MODBUS_OK;
}
```


## نحوه پیاده سازی تابع پردازش فریم  MODBUS_FARME_PROCESS

این تابع جهت آماده سازی پاسخ به مستر و انجام عملیات مربوطه (نوشتن/خواندن در رجیسترها یا کویل ها) با توجه به فانکشن¬کد می باشد.

### آماده سازی فریم پاسخ به مستر برای فانکشن¬کد 01 Read Coil Status

وضعیت کویل در پیام پاسخ به عنوان یک کویل در هر بیت از فیلد داده بسته بندی می شود.
پیام کوئری، شروع کویل و تعداد کویل های خوانده شده را مشخص می کند. 

**نکته**:آدرس کویل ها از صفر شروع می شود: کویل های 1-16 به صورت 0-15 نشان داده می شوند.

چون باید بیت های مربوط به آرایه ای از بایت ها رو استخراج کنیم
ابتدا ایندکس شروع آرایه کویل ها رو محاسبه  می کنیم. اگر آدرس کویل شروع را بر هشت تقسیم کنیم شروع ایندکس آرایه بدست می آید و برای محاسبه بیت شروع آن هم، باقی مانده تقسیم بر هشت را حساب می کنیم. 

```C

 /* 1. Get coil value from COIL_MEM. note: coils 1–16 are addressed as 0–15 (coil_counter % 8) */
        coil = (COIL_MEM[(coil_counter / 8)] >> coil_counter % 8) & 1;
```


حالا یک حلقه ایجاد می کنیم و صفر یا یک بودن بیت های آرایه مربوط به کویل را می خوانیم.

```C

/* 2. Set coil in response (in bit located) */
        if (coil == 1)
            Constructed_ResponseFrame[array_byte] |= (1 << bit);
        else
            Constructed_ResponseFrame[array_byte] &= ~(1 << bit);
            
```

1. get coil value from mem
2. set coli value in reponse (in bit located)
2. 1. bit 0 - 7 then plus array byte


```C

#if MAX_COIL > 0
/**
 * @brief Reads the ON/OFF status of discrete outputs in the slave.
 * maximum parameters supported by various controller models:
 * 184/384:[800 coil]   484:[512 coil]     584/984/884:[2000 coil]    M84:[64 coil]
 *
 * @param RequestFrame
 * @param Constructed_ResponseFrame
 * @return return Frame length
 */
static unsigned char SLAVE_Read_Coil_Status_Operation(
    unsigned char *RequestFrame, unsigned char *Constructed_ResponseFrame)
{
    unsigned int Start_address = RequestFrame[2] << 8 | RequestFrame[3];
    unsigned int Num_of_coil = RequestFrame[4] << 8 | RequestFrame[5];
    unsigned int Start_Coil = Start_address;

    if (Start_address + Num_of_coil > MAX_COIL)
        return Modbus_Exception(ILLEGAL_DATA_ADDRESS, Constructed_ResponseFrame);

    /* Constructing the response frame to the master */
    Constructed_ResponseFrame[0] = RequestFrame[0]; /* Slave Address */
    Constructed_ResponseFrame[1] = RequestFrame[1]; /* Function Code */

    unsigned int Byte_Count = 0; /*< [unsigned char] For 2000 coil, Maximum byte: 250 + 5 = 255 */
    unsigned int array_byte = 3; /*< [unsigned char] For 2000 coil, Maximum byte: 250 + 3 = 253 */

    unsigned int coil_counter = Start_Coil;
    unsigned char coil = 0;
    char bit = 0;
    int Coil_len = Num_of_coil;

    while (Coil_len--)
    {
        if (bit == 0)
            Byte_Count++;

        /* 1. Get coil value from COIL_MEM. note: coils 1–16 are addressed as 0–15 (coil_counter % 8) */
        coil = (COIL_MEM[(coil_counter / 8)] >> coil_counter % 8) & 1;

        /* 2. Set coil in response (in bit located) */
        if (coil == 1)
            Constructed_ResponseFrame[array_byte] |= (1 << bit);
        else
            Constructed_ResponseFrame[array_byte] &= ~(1 << bit);

        /* 2.1 bit 0 - 7 then plus array byte */
        if (bit++ == 7)
        {
            bit = 0;
            array_byte++;
        }

        coil_counter++;
    }

    Constructed_ResponseFrame[2] = Byte_Count; /* Byte Count */

    return 3 + Byte_Count;
}
#endif
```


### آماده سازی فریم پاسخ به مستر برای فانکشن¬کد 02 Read Input Status

```C

#if MAX_INPUT > 0
/**
 * @brief Reads the ON/OFF status of discrete inputs in the slave.
 * maximum parameters supported by various controller models:
 * 184/384:[800 coil]   484:[512 coil]     584/984/884:[2000 coil]    M84:[64 coil]
 *
 * @param RequestFrame
 * @param Constructed_ResponseFrame
 * @return return Frame length
 */
static unsigned char SLAVE_Read_Input_Status_Operation(
    unsigned char *RequestFrame, unsigned char *Constructed_ResponseFrame)
{
    unsigned int Start_address = RequestFrame[2] << 8 | RequestFrame[3];
    unsigned int Num_of_Input = RequestFrame[4] << 8 | RequestFrame[5];
    unsigned int Start_Input = Start_address;

    if (Start_address + Num_of_Input > MAX_INPUT)
        return Modbus_Exception(ILLEGAL_DATA_ADDRESS, Constructed_ResponseFrame);

    /* Constructing the response frame to the master */
    Constructed_ResponseFrame[0] = RequestFrame[0]; /* Slave Address */
    Constructed_ResponseFrame[1] = RequestFrame[1]; /* Function Code */

    unsigned int Byte_Count = 0; /*< [unsigned char] For 2000 input, Maximum byte: 250 + 5 = 255 */
    unsigned int array_byte = 3; /*< [unsigned char] For 2000 input, Maximum byte: 250 + 3 = 253 */

    unsigned int Input_counter = Start_Input;
    unsigned char Input = 0;
    char bit = 0;
    int Input_len = Num_of_Input;

    while (Input_len--)
    {
        if (bit == 0)
            Byte_Count++;

        /* 1. Get Input value from INPUT_MEM. note: Inputs 1–16 are addressed as 0–15 (Input_counter % 8) */
        Input = (INPUT_MEM[(Input_counter / 8)] >> Input_counter % 8) & 1;

        /* 2. Set input in response (in bit located) */
        if (Input == 1)
            Constructed_ResponseFrame[array_byte] |= (1 << bit);
        else
            Constructed_ResponseFrame[array_byte] &= ~(1 << bit);

        /* 2.1 bit 0 - 7 then plus array byte */
        if (bit++ == 7)
        {
            bit = 0;
            array_byte++;
        }

        Input_counter++;
    }

    Constructed_ResponseFrame[2] = Byte_Count; /* Byte Count */

    return 3 + Byte_Count;
}
#endif
```

### آماده سازی فریم پاسخ به مستر برای فانکشن¬کد 03 Read Holding Registers

```C

#if MAX_HOLDING_REGISTERS > 0
/**
 * @brief Reads the binary contents of holding registers in the slave.
 * maximum parameters supported by various controller models:
 * 184/384:[100 holding registers]   484:[254 holding registers]
 * 584/984/884:[125 holding registers]    M84:[64 holding registers]
 *
 * @note Note: Data is scanned in the slave at the rate of 125 registers per scan for
 *  984–X8X controllers (984–685, etc), and at the rate of 32 registers per scan for
 * all other controllers. The response is returned when the data is completely assembled
 * @param RequestFrame
 * @param Constructed_ResponseFrame
 * @return return Frame length
 */
static unsigned char SLAVE_Read_Holding_Registers_Operation(
    unsigned char *RequestFrame, unsigned char *Constructed_ResponseFrame)
{
    unsigned int Start_address = RequestFrame[2] << 8 | RequestFrame[3];
    unsigned int Num_of_Holding_Registers = RequestFrame[4] << 8 | RequestFrame[5];
    unsigned int Start_Holding_Registers = Start_address;

    if (Start_address + Num_of_Holding_Registers > MAX_HOLDING_REGISTERS)
        return Modbus_Exception(ILLEGAL_DATA_ADDRESS, Constructed_ResponseFrame);

    /* Constructing the response frame to the master */
    Constructed_ResponseFrame[0] = RequestFrame[0]; /* Slave Address */
    Constructed_ResponseFrame[1] = RequestFrame[1]; /* Function Code */

    unsigned int Byte_Count = Num_of_Holding_Registers * 2; /* Byte Count */

    int Holding_Registers_len = Num_of_Holding_Registers;
    unsigned int Holding_Registers_count = Start_Holding_Registers;
    unsigned int byte = 3;

    while (Holding_Registers_len--)
    {
        Constructed_ResponseFrame[byte++] =
            HOLDING_REGISTERS_MEM[Holding_Registers_count] >> 8; /* Hi Holding Registers */

        Constructed_ResponseFrame[byte++] = 0x00ff & HOLDING_REGISTERS_MEM[Holding_Registers_count]; /* Lo Holding Registers */

        Holding_Registers_count++;
    }

    Constructed_ResponseFrame[2] = Byte_Count; /* Byte Count */

    return 3 + Byte_Count;
}
#endif
```

## پیاده سازی تایم اوت یا مهلت زمانی برای تابع مانیتور شبکه مدباس در محیط cubeide برای میکروکنترلرهای STM32

```C

	/* Init tickstart for timeout management */
	tickstart_for_monitor_timeout = *Tick;
	while (1) {
		/* 0. Check for monitor function timeout */
		currenttick_for_monitor_timeout = *Tick;
		if (currenttick_for_monitor_timeout - tickstart_for_monitor_timeout > monitor_fun_timeout)
			return MODBUS_MONITOR_TIMEOUT;
```

## نحوه فراخوانی توابع مانیتور شبکه مدباس و پردازش، در بدنه اصلی برنامه


```C

	while (1) {

		ModbusStatus_t res = MODBUS_RTU_MONITOR(buff, 1800, receive_uart,
				&uwTick);

		if (res == MODBUS_OK) {
			uint16_t len = MODBUS_FARME_PROCESS(buff, response);

			HAL_UART_Transmit(&huart1, response, len, 100);
		} else if (res == MODBUS_MONITOR_TIMEOUT) {
			HAL_UART_Transmit(&huart1, (uint8_t*) "MODBUS MONITOR TIMEOUT", 22,
					100);
		}

		/* USER CODE END WHILE */

		/* USER CODE BEGIN 3 */
	}
```

## تابع کال بک خواندن از یوارت

تعریف توابع کالبک دریافت یوارت در کتابخانه مدباس به صورت weak تا کاربر این توابع را برای پیاده سازی در برنامه خود استفاده کند. 


```C

/**
 * @brief
 * NOTE : This function should not be modified, when the callback is needed,
 the uart_receive could be implemented in the user file
 * @return __weak
 */
__attribute__((weak))   ModbusStatus_t uart_receive(uint8_t *Data) {
	return 0;
}
```

Example:

```C

/* USER CODE BEGIN PFP */
ModbusStatus_t uart_receive(uint8_t *Data) {
	ModbusStatus_t res ;
	res = (ModbusStatus_t) HAL_UART_Receive(&huart1, Data, 1, 3000);
	return res;
}
```

بررسی  crc انجام شود که اگر یکی نبود بافر را پاک کند و دوباره تابع مانیتور شبکه را فراخوانی کند.

```C

/* Check CRC calculated, that is equal with CRC frame */
	if (calculate_crc != frameCRC) {
		/* clear buffer */
		while(starting_address_of_buffer < mbus_frame_buffer){
			*mbus_frame_buffer-- = 0x00;
		}
		/* return to 0. */
		MODBUS_RTU_MONITOR(starting_address_of_buffer, monitor_fun_timeout, Tick);
		//return MODBUS_ERROR;
	}
```

## پیاده سازی مهلت زمانی بین فریم ها 
تا زمانی که در داخل حلقه بایتی از یوارت دریاقت میشود پس شبکه در حال ردوبدل داده هست و سکوت برقرار نیست. اگر دریافت نداشت و مقدار برگشتی آن اررور مهلت زمانی بود از حلقه خارج شود، در نتیجه می توان از بایت نخست فریم جدید را دریافت کرد. برای اینکار  در یک حلقه منتظر می مانیم تا یک بایت دریافت شود دقت داشته باشید که ر این حلقه باید مهلت زمانی تابع بررسی شود.

```C
		/* frame timeout management */
		do {
			res = (*receive_uart_fun)(&rec_byte);..
		} while (res == MODBUS_OK);


		/* 2. Ready for receive of first Byte (Address Field) */
		do {
			res = (*receive_uart_fun)(&rec_byte);
			currenttick_for_monitor_timeout = *Tick;
			if (currenttick_for_monitor_timeout - tickstart_for_monitor_timeout
					> monitor_fun_timeout)
				return MODBUS_MONITOR_TIMEOUT;
		} while (res != MODBUS_OK);
```

**خطاهایی که در حلقه مهلت زمانی تابع مانیتور رخ می دهد و باید به ابتدای حلقه برگردد**

1.	آدرس دستگاه کنونی نیست.
2.	مهلت زمانی دریافت بایت رخ دهد یعنی بین بایت های دریافتی فاصله ایجاد شود.
3.	crc یکی نباشد.
برای موارد 2 و 3 باید بافر پاک شود بعد continue شود.


**به موارد بالا سریز بافر برای بررسی اضافه شود**

**پین دایرکشن**

برای مد RS-485 باید یک پین به عنوان دایرکشن انتقال داده در نظر گرفته شود. برای ارسال، منطق یک و برای دریافت، منطق صفر شود.

**تا به اینجا تابع مانیتور شبکه مدباس در بدنه اصلی برنامه فراخوانی می شود و در آن**

1.	در حلقه وایل مهلت زمانی مانیتور بررسی میشود.
2.	تا زمانی که بایت دریافت می کند در یک حلقه باشد تا مهلت زمانی برای دریافت بایت رخ دهد. (یعنی اگر از قبل فریمی در حال ارسال است صبر کنیم تا تمام شود)
3.	بعد از مرحله 2 برای دریافت اولین بایت در یک حلقه صبر کنیم تا بایت دریافت شود در این حلقه مهلت زمانی مانیتور شبکه بررسی شود.
4.	در صورتی که آدرس در محدوده مجاز یا هم آدرس ما یا پخش همگانی نبود به شماره 1 برگردد.
5.	بایت دریافتی (آدرس) را در بافر ذخیره کند و crc محاسبه شود.
نکته: از شماره 5 به بعد هر بار که یک بایت دریافت می شود نباید مهلت زمانی دریافت بایت رخ دهد واگر رخ دهد باید به شماره 1 برگردد. بنابراین قبل از ذخیره در بافر و محاسبه crc مهلت زمانی بررسی شود.
6.	بایت فانکشن رو دریافت و براساس آن تعداد بایت هایی که بعد از آن باید دریافت کند را مشخص کند
7.	بعد از دریافت بایت های قبل از crc براساس فانکشن¬کد دو بایت crc دریافت شود
8.	crc محاسبه شده با crc  دریافتی از فریم مقایسه شود اگر یکی نبو د بافر پاک شودو به شماره 1 برگردد.
9.	فریم دریافتی برای پردازش به تابع پردازش جهت اعمال عملیات و آماده سازی فریم پاسخ به مستر ارسال می شود.
10.	فریم پاسخ آماده شده توسط مرحله قبل ارسال شود.


## پیاده سازی تابع آماده سازی فریم پاسخ به مستر برای فانکشن¬کد 15 (0F Hex) Force Multiple Coils

دوبایت دوبایت انجام می شود
بایت اولی که ارسال شده کویل های پایین تر هستند (بیت پایین کویل پایین تر)
بایت بعدی که ارسال شده کویل های بالاتر می شوند
مثلا اگر 4 بایت بود و کویل شروع 27 بود میشود
بایت اول، کویل های 27-33
بایت دوم، کویل های 34-40

به کمک آدرس شروع و تعداد کویل ها انجام می دهیم


```C

#if MAX_COIL > 0
/**
 * @brief Forces each coil in a sequence of coils to either ON or OFF.
 *
 * @param RequestFrame
 * @param Constructed_ResponseFrame
 * @return return Frame length
 */
static unsigned char SLAVE_Force_Multiple_Coils_Operation(
    unsigned char *RequestFrame, unsigned char *Constructed_ResponseFrame)
{
    unsigned int Start_address = RequestFrame[2] << 8 | RequestFrame[3];
    unsigned int Quantity_of_Coils = RequestFrame[4] << 8 | RequestFrame[5];

    if (Start_address + Quantity_of_Coils > MAX_COIL)
        return Modbus_Exception(ILLEGAL_DATA_ADDRESS, Constructed_ResponseFrame);

    unsigned int coil_counter = Start_address;
    char bit = 0;
    unsigned int Byte_Counter = 0;
    unsigned char coil;

    /* Write */
    while (coil_counter < Quantity_of_Coils + Start_address)
    {
        /* 1. Get coil value from data frame. */
        coil = (RequestFrame[7 + Byte_Counter] >> bit % 8) & 1;

        /* 2. Set coli's in COIL_MEM (in bit located) */
        if (coil == 1)
            COIL_MEM[coil_counter / 8] |= (1 << coil_counter % 8);
        else
            COIL_MEM[coil_counter / 8] &= ~(1 << coil_counter % 8);

        /* 2.1 bit 0 - 7 then plus array byte */
        if (bit++ == 7)
        {
            bit = 0;
            Byte_Counter++;
        }

        coil_counter++;
    }

    /* The normal response returns the slave address, function code, starting address,
     and quantity of coils forced. */
    Constructed_ResponseFrame[0] = RequestFrame[0]; /* Slave Address */
    Constructed_ResponseFrame[1] = RequestFrame[1]; /* Function Code */
    Constructed_ResponseFrame[2] = RequestFrame[2]; /* Coil Address Hi */
    Constructed_ResponseFrame[3] = RequestFrame[3]; /* Coil Address Lo */
    Constructed_ResponseFrame[4] = RequestFrame[4]; /* Quantity of Coils Hi */
    Constructed_ResponseFrame[5] = RequestFrame[5]; /* Quantity of Coils Lo */

    return 6; //
}
#endif
```

## پیاده سازی تابع آماده سازی فریم پاسخ به مستر برای فانکشن¬کد  16 (10 Hex) Preset Multiple Registers 

```C

#if MAX_HOLDING_REGISTERS > 0
/**
 * @brief Presets values into a sequence of holding registers. When broadcast,
 * the function presets the same register references in all attached slaves.
 *
 * @param RequestFrame
 * @param Constructed_ResponseFrame
 * @return return Frame length
 */
static unsigned char SLAVE_Preset_Multiple_Register_Operation(
    unsigned char *RequestFrame, unsigned char *Constructed_ResponseFrame)
{
    unsigned int Start_address = RequestFrame[2] << 8 | RequestFrame[3];
    unsigned int Number_of_Registers = RequestFrame[4] << 8 | RequestFrame[5];

    if (Start_address + Number_of_Registers > MAX_HOLDING_REGISTERS)
        return Modbus_Exception(ILLEGAL_DATA_ADDRESS, Constructed_ResponseFrame);

    unsigned int registers_counter = Start_address;
    unsigned int Byte_Counter = 7;

    /* Write */
    while (registers_counter < Number_of_Registers + Start_address)
    {
        HOLDING_REGISTERS_MEM[registers_counter] = RequestFrame[Byte_Counter++] << 8;
        HOLDING_REGISTERS_MEM[registers_counter] |= RequestFrame[Byte_Counter++];
        registers_counter++;
    }

    /*
     The normal response returns the slave address, function code, starting address,
     and quantity of registers preset. */
    Constructed_ResponseFrame[0] = RequestFrame[0]; /* Slave Address */
    Constructed_ResponseFrame[1] = RequestFrame[1]; /* Function Code */
    Constructed_ResponseFrame[2] = RequestFrame[2]; /* Coil Address Hi */
    Constructed_ResponseFrame[3] = RequestFrame[3]; /* Coil Address Lo */
    Constructed_ResponseFrame[4] = RequestFrame[4]; /* No. of Registers Hi */
    Constructed_ResponseFrame[5] = RequestFrame[5]; /* No. of Registers Lo */

    return 6;
}
#endif
```

**1.	کانفیگ حافظه وضعیت کویل ها، وضعیت ورودی ها، رجیسترهای نگه¬دارنده و رجیسترهای ورودی:**

**2.	کانفیگ نوع مانیتور شبکه مدباس:**

```C

typedef enum  {
	Listen_Only = 0,
	Normal = 1
} ModbusMonitorMode_t;

```
**عملیات نوشتن/خواندن روی چه مواردی انجام میشود:**

1.	وضعیت سیم پیچ ها (کویل ها)
2.	وضعیت ورودی ها
3.	رجیسترهای نگه دارنده
4.	رجیسترهای ورودی


## نحوه استفاده از کتابخانه مدباس  (ورژن فعلی)

**توجه**: این کتابخانه برای میکروکنترلر استفاده می شود.

```C

set_slave_ID(17);
```

### راه اندازی ارتباط سریال 

1.	شماره پورت (آدرس رجیستر یوارت در میکروکنترلر)
2.	مقداردهی باودریت
3.	پریتی
4.	استاپ بیت


```C
Serial_t default_serial;
default_serial.UART = (uint32_t*) USART1;
default_serial.BaudRate = 9600;
default_serial.Parity = NONE_PARITY;
default_serial.StopBit = StopBit_1;

modbus_serial_init(&default_serial);

```

### تنظیم حافظه برای رجیستر های(نگه دارنده و ورودی) و کویل ها و ورودی های مجزا 

تنظیم میزان فضای ذخیره برای 
•	کویل (درصورت استفاده کاربر)
•	ورودی مجزا (درصورت استفاده کاربر)
•	رجیستر نگه دارنده (درصورت استفاده کاربر)
•	رجیستر ورودی (درصورت استفاده کاربر)

برای این کار در کتابخانه modbus.h دیفاین های زیر را در این فایل با توجه به نیاز برنامه خود تغییر دهید.

```C

/* Modbus memory map for COIL, INPUT, HOLDING_REGISTERS, INPUT_REGISTERS */
#define MAX_COIL      				8 	/*< 184/384:[800]   484:[512]     584/984/884:[2000]    M84:[64] */
#define MAX_INPUT      				0 		/*< 184/384:[800]   484:[512]     584/984/884:[2000]    M84:[64] */
#define MAX_HOLDING_REGISTERS       8 	/*< 184/384:[100]   484:[254]     584/984/884:[125]    M84:[64] */
#define MAX_INPUT_REGISTERS         0 		/*< 184/384:[100]   484:[32]     584/984/884:[125]    M84:[4] */

```

**نکته**: در زمان اعمال دستورات مستر، باید به حداگثر تعداد (کویل، رجیستر و...) توجه کرد و در صورت نیاز پاسخ استثنا آماده کرد.


1.	اول باید بررسی شود که آیا در این دستگاه حافظه ای برای آن در نظر گرفته شده است یا نه؟
2.	دوم اگر هم حافظه ای هم موجود بود نباید بیشتر از تعداد آن برای خواندن و یا نوشتن از سمت مستر درخواست شود. 

در صورتی که موارد بالا نقض شد پاسخ استثنا تولید شود. (به عنوان مثال، اگر درخواست خواندن یک کویل یا رجیستر ناموجود باشد)، Slave یک پاسخ استثنایی را برمی‌گرداند که به مستر از ماهیت خطا اطلاع می‌دهد.


## آماده سازی پاسخ استثنا برای مستر

```C

/**
 * @brief
 *
 * @param Modbus_Exception
 * @param ResponseFrame
 * @return return Frame length
 */
unsigned char Modbus_Exception(Modbus_Exception_Code_t Modbus_Exception_Code, unsigned char *ResponseFrame)
{
    ResponseFrame[0] = SLAVE_ADDRESS;         /* Slave Address */
    ResponseFrame[1] = 0x81;                  /* Function */
    ResponseFrame[2] = Modbus_Exception_Code; /* Exception Code */

    return 3;
}
```


سپس در برنامه از تابع مانیتور شبکه مدباس استفاده میکنیم

این تابع با توجه به تنظیمات بالا فریم مربوط به این دستگاه را از سریال دریافت می کند، پردازش می کند و دستورات را اعمال می کند(خواندن/نوشتن روی رجیسترها یا کویل ها)و پاسخ و یا پاسخ استثنا را برای مستر آماده می کند و از طریق سریال ارسال می کند.

```C

/* Modbus network monitor to receive frames from the master device */
ModbusStatus_t res = MODBUS_RTU_MONITOR(buff, 3000, &uwTick, Normal);

```

**بعد از این تابع، کاربر می تواند مقادیر حافظه رجیستر ها و کویل ها را با مقادیر قبلی مقایسه کند (در صورتی که از سوی مستر روی حافظه دستور نوشتن صورت گرفته باشد) اگر تغییری وجود داشت، روی خروجی های برنامه مثلا رله ها اعمال شود.**

## نحوه استفاده از توابع Handler (نسخه فعلی)

توابع Handler را از کتابخانه modbus_handler.c کپی کنید و از آن در فایل خود برای پیاده سازی استفاده کنید.

```C

/**
 * @file modbus_handler.h
 * @author Mehdi Adham (mehdi.adham@yahoo.com)
 * @brief This library Handle the Modbus Communication.
 * @version 0.1
 * @date 2022-08-29
 *
 * @copyright Copyright (c) 2022
 *
 */


#ifndef __MODBUS_HANDLER_H
#define __MODBUS_HANDLER_H

/**
 * @brief
 * NOTE : This function should not be modified, when the callback is needed,
 the uart_receive Handler could be implemented in the user file
 * @return Modbus Status
 */
__attribute__((weak))	  ModbusStatus_t modbus_uart_receive_Handler(uint8_t *Data) {
	return 0;
}

/**
 * @brief
 * NOTE : This function should not be modified, when the callback is needed,
 the uart transmit Handler could be implemented in the user file
 * @return None
 */
__attribute__((weak)) void modbus_uart_transmit_Handler(uint8_t *Data,
		uint16_t length) {

}

/**
 * @brief
 * NOTE : This function should not be modified, when the callback is needed,
 the uart_init Handler could be implemented in the user file
 * @return Modbus Status
 */
__attribute__((weak)) void modbus_uart_init_Handler(Serial_t *Serial) {

}


#endif
```

**نمونه ای از پیاده سازی Handler برای کتابخانه HAL (STM32)**

```C

ModbusStatus_t modbus_uart_receive_Handler(uint8_t *Data) {
	ModbusStatus_t res;
	res = (ModbusStatus_t) HAL_UART_Receive(&huart1, Data, 1, timeout_3_5C);
	return res;
}

void modbus_uart_transmit_Handler(uint8_t *Data, uint16_t length) {
	HAL_Delay(5);
	HAL_UART_Transmit(&huart1, Data, length, 100);
}

void modbus_uart_init_Handler(Serial_t *Serial) {
	huart1.Instance = (USART_TypeDef*) Serial->UART;
	huart1.Init.BaudRate = Serial->BaudRate;

	if (Serial->StopBit == StopBit_1)
		huart1.Init.StopBits = UART_STOPBITS_1;
	else
		huart1.Init.StopBits = UART_STOPBITS_2;

	if (Serial->Parity == NONE_PARITY)
		huart1.Init.Parity = UART_PARITY_NONE;
	else if (Serial->Parity == ODD_PARITY)
		huart1.Init.Parity = UART_PARITY_ODD;
	else
		huart1.Init.Parity = UART_PARITY_EVEN;

	huart1.Init.WordLength = UART_WORDLENGTH_8B;
	huart1.Init.Mode = UART_MODE_TX_RX;
	huart1.Init.HwFlowCtl = UART_HWCONTROL_NONE;
	huart1.Init.OverSampling = UART_OVERSAMPLING_16;

	HAL_UART_Init(&huart1);
}
```



## تغییر پین DIR

 برای پین دایرکشن آی سی 485 نیاز به تابع هندلر نیست در تابع هندلر ارسال یوارت، قبل و بعد از ارسال پین را تغییر می دهیم:

```C

void modbus_uart_transmit_Handler(uint8_t *Data, uint16_t length) {
	HAL_Delay(5);
	HAL_GPIO_WritePin(DIR_GPIO_Port, DIR_Pin, GPIO_PIN_SET);
	HAL_UART_Transmit(&huart1, Data, length, 100);
	HAL_GPIO_WritePin(DIR_GPIO_Port, DIR_Pin, GPIO_PIN_RESET);
}
```

**نکته: سرریز بافر دریافت فریم در تابع مانیتور بررسی شود حتما. حداکثر 256 بایت است. بیشتر از آن بافر پاک شود و از تابع خارج شود. با مقدار برگشتی وضعیت مانیتور سریز بافر**


**آدرس 0 پخش همگانی تست شود نباید پاسخ داده شود.**