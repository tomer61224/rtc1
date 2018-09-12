//  נתונים השייכים לרכיב השעון
#define sda  20  //  הדק הנתון הטורי של השעון
#define scl  21  //  הדק פולסי השעון מהמיקרו אל השעון
bool second_passed;    //  ביט שמראה שעברה שנייה

//---------   נתונים לרכיב השעון  ----------------------------------
unsigned char day[8] = {0, 'א', 'ב', 'ג', 'ד', 'ה', 'ו', 'ש'};
unsigned char seconds = 50;
unsigned char minutes = 0x6;
unsigned char hours = 0x19;
unsigned char week_day = 1;        //  מראה את היום בשבוע  -  א' עד שבת
unsigned char month = 0x1;
unsigned char year = 0x16;
unsigned char month_day = 0x10;   //  מראה את היום בחודש
unsigned char read_data;      // הנתון שקוראים מהשעון

void setup(){
  pinMode(scl, OUTPUT); //  הדק השעון כהדק יציאה
  pinMode(sda, OUTPUT);
  pinMode(2, OUTPUT);

  Serial.begin(9600);  //  קצב תקשורת טורית עם מסך המחשב - 9600 ביטים בשנייה
  Serial.println("Tomer's Clock");
  init_rtc();
}


void loop()
{
  clock();   

   if(minutes == 0x6)
   {
    digitalWrite(2, HIGH);
    digitalWrite(2, LOW);
   }
   else 
    {  
    digitalWrite(2, LOW);
    }

}


//פונקציה הבודקת אם השעון מאותחל. אם לא היא מכניסה נתוני זמן ותאריך אקראיים
void init_rtc()
{
  unsigned char j;
  read_rtc(0);
  if (read_data >= 0x59)
 {
    write_rtc(0, 0);
    write_rtc(minutes, 1);
    write_rtc(hours, 2);
    write_rtc(week_day, 3);
    write_rtc(month_day, 4);
    write_rtc(month, 5);
    write_rtc(year, 6);
    write_rtc(0, 7);
  }
}

//-------  תוכנית המטפלת בשעון ------------
void clock()
{
  

  read_rtc(0);  //  קרא מהשעון את השניות
  if (read_data != seconds) // אם השניות שקראנו מהשעון לא שוות למה שיש במשתנה  המכיל את השניות , כלומר עברה שנייה
  {
    second_passed = 1;
    seconds = read_data;  //השניות השמת השניות מהשעון במשתנה
    read_rtc(1);    // קריאת הדקות מהשעון
    minutes = read_data;  // השמת הדקות מהשעון במשתנה הדקות
    read_rtc(2);    // קריאת השעות מהשעון
    hours = read_data;  // השמת השעות מהשעון במשתנה השעות
    read_rtc(3);    // קריאת השעות מהשעון
    week_day = read_data; // השמת השעות מהשעון במשתנה השעות
    read_rtc(4);    // היום בחודש
    month_day = read_data;
    read_rtc(5);    // קריאה של החודש
    month = read_data;
    read_rtc(6);    // קריאה של השנה
    year = read_data;
    


   }

}
//--------- I2C  פונקציה לשליחת 8 ביטים בתקשורת -----------
void _8bits(unsigned char data8)
{
  pinMode(sda, OUTPUT);
  for (unsigned char i = 0; i <= 7; i++)
  {
    digitalWrite(sda, data8 / 128);
    data8 = data8 << 1;
    digitalWrite(scl, 1);
    delayMicroseconds(10);
    digitalWrite(scl, 0);
    delayMicroseconds(10);
  }
}

//-------  פונקציה הקוראת 8 ביטים מרכיב השעון ----------
void read_sda() //פונקצית קריאת תוכן בית מהשעון
{
  pinMode(sda, INPUT); // הפיכת הדק הנתון לקלט
  digitalWrite(scl, 0);
  read_data = 0;
  for (unsigned char i = 0; i < 8; i++) //   לולאה של 8 פעמים
  {
    digitalWrite(scl, 1);
    delayMicroseconds(10);
    digitalWrite(scl, 0);
    read_data = read_data << 1; //הזזת תוכן שמאלה מקום 1
    read_data = read_data + digitalRead(sda); //הוספת ביט המידע לתוכן
    delayMicroseconds(10);
  }
  pinMode(sda, OUTPUT);
}

//----  שליחת ביט התחלה ליצירת תקשורת עם רכיב השעון ----
//    שינוי במצב קו הנתון מגבוה לנמוך כאשר השעון נמצא בגבוה מוגדר כמצב START
void start()
{
  digitalWrite(scl, 1);
  delayMicroseconds(10);
  digitalWrite(sda, 1);
  delayMicroseconds(10);
  digitalWrite(sda, 0);
  delayMicroseconds(10);
  digitalWrite(scl, 0);
}

//--- שליחת ביט סיום תקשורת לרכיב השעון -----
//שינוי במצב קו הנתון מנמוך לגבוה כאשר השעון במצב גבוה מוגדר כמצב  STOP .
void stop_rtc()
{
  digitalWrite(sda, 0);
  delayMicroseconds(10);
  digitalWrite(scl, 1);
  delayMicroseconds(10);
  digitalWrite(sda, 1);
}

//-----  המתנה לקבלת אישור מרכיב השעון -------
/*
   רכיב היוצר  ACKNOLEDGE חייב להוריד את קו הנתון הטורי – SDA – ל 0 בזמן פולס השעון
   , כלומר שקו הנתון יהיה יציב בנמוך בזמן שקו השעון בגבוה.
*/
void ack()
{
  pinMode(sda, INPUT);
  digitalWrite(scl, 1);
  while (digitalRead(sda)) ;
  digitalWrite(scl, 0);
  delay(1);
  pinMode(sda, OUTPUT);
}

//--- STOP קוראת לפונקציה השולחת את הביית, ממתינה לאישור ושולחת START פונקציה השולחת   -----
void write_rtc(unsigned char data_wr, unsigned char address)    //פונקצית כתיבה לזיכרון
{
  start();      //פונקצית התחל
  _8bits(0xd0);//פוקציה הוצאה טורית פקודת כתיבה +3 ביטים עליונים של הכתובת
  ack();  // המתנה לאישור מהרכיב
  _8bits(address);//פונקציה הוצאה טורית פקודת האומרת לאיזו כתובת לכתוב את הנתון
  ack();      //פונקצית המתנה למענה
  _8bits(data_wr);  //פונקציה הוצאה טורית 8 ביטים של התוכן
  ack();    //פונקצית המתנה למענה
  stop_rtc();   //פונקצית עצור
  delayMicroseconds(10);
}
//ACK  ממתינה לאישור שולחת את הכתובת הרצויה, ממתינה לאישור,שולחת פקודת קריאה,קוראת את הנתון ושולחת START פונקציה השולחת
void read_rtc(unsigned char address)  //פונקצית קריאה מזיכרון
{
  start();    //פונקצית התחל
  _8bits(0xd0);//פוקציה הוצאה טורית פקודת כתיבה +3 ביטים עליונים של הכתובת
  ack();    //פונקצית המתנה למענה
  _8bits(address);//פונקציה הוצאה טורית 8 ביטים תחתונים של הכתובת
  ack();    //פונקצית המתנה למענה
  start();    //פונקצית התחל
  _8bits(0xd1);//פוקציה הוצאה טורית פקודת קריאה +3 ביטים עליונים של הכתובת
  read_sda();   //פונקצית קריאת תוכן מהזיכרון
  //---------   שליחת אישור לרכיב השעון------------
  digitalWrite(sda, 1);
  delayMicroseconds(10);
  digitalWrite(scl, 1);
  delayMicroseconds(10);
  digitalWrite(scl, 0);
  stop_rtc();   //פונקצית עצור


  
}













