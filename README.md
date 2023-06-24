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
| 1 | 01 Read Coil Status  <br> 02 Read Input Status <br>03 Read Holding Registers <br> 04 <br>Read Input Registers <br>05 Force Single Coil<br>06 Preset Single Register <td dir="rtl">اینجا بعد از فانکشن مشخصا 4 بایت دیگر دریافت می کند بدون احتساب دو بایت CRC</td>
| 2 | 07 Read Exception Status 11 (0B Hex) Fetch Comm Event Counter<br>12 (0C Hex) Fetch Comm Event Log<br>17 (11 Hex) Report Slave ID <td dir="rtl">اینجا بعد از فانکشن CRC می آید.</td>
| 3 | 15 (0F Hex) Force Multiple Coils <br> 16 (10 Hex) Preset Multiple Registers | بایت هفتم تعداد بایت های بعد از خودش را مشخص می کند بدون احتساب دو بایت CRC |
| 4 | 20 (14Hex) Read General Reference <br> 21 (15Hex) Write General Reference | بایت سوم تعداد بایت های بعد از خودش را مشخص می کند بدون احتساب دو بایت CRC |
| 5 | 22 (16Hex) Mask Write 4X Register | اینجا بعد از فانکشن مشخصا 6 بایت دیگر دریافت می کند بدون احتساب دو بایت CRC |
| 6 | 23 (17Hex) Read/Write 4X Registers | بایت یازدهم تعداد بایت های بعد از خودش را مشخص می کند بدون احتساب دو بایت CRC |
| 7 | 24 (18Hex) Read FIFO Queue | اینجا بعد از فانکشن مشخصا 2 بایت دیگر دریافت می کند بدون احتساب دو بایت CRC |


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


