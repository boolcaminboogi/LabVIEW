# LabVIEW
폴리텍 마이크로프로세서 응용 실습 labVIEW실습 과제들 (김성곤 교수님)



https://github.com/boolcaminboogi/LabVIEW/assets/128253251/32066f76-26af-49c8-b02e-81b78a2acb1b

[Uploading SWITCH COUNTER.c…]()


https://github.com/boolcaminboogi/LabVIEW/assets/128253251/1549580e-b3f7-4440-817d-65fa2a9c0e60

[코드]
#define F_CPU 16000000UL // 동작 클럭 지정

#include <avr/interrupt.h> // 인터럽트 헤더 파일
#include <avr/io.h> // I/O 헤더 파일
#include <util/delay.h> // 딜레이 헤더 파일

volatile unsigned int N1=0, N10=0, N100=0, N1000=0; //각 FND의 값 저장 변수
int number[10] = {0x40, 0x79, 0x24, 0x30, 0x19, 0x12, 0x02, 0x58, 0x00, 0x18};  //0, 1, 2, 3, 4, 5, 6, 7, 8, 9
volatile unsigned char count = 0, AMPM = 0, timer0_flag = 0;
int pos = 0;

// 포트 지정
#define DIG1 PA0			//다이오드 포트 1번째
#define DIG2 PA1			//다이오드 포트 2번째
#define DIG3 PA2			//다이오드 포트 3번째
#define DIG4 PA3			//다이오드 포트 4번째
#define DIGAMPM PA4			//AM,PM 포트
#define DIGD1D2 PA5			// : 설정 포트

#define D1D2 PC0		// :포트
#define AM PC1			//AM포트
#define PM PC2			//PM포트

#define BUZ PC3			//알람 포트

// SCD-447A 초기 점검z 함수
void init_seg(void)
{
	PORTF = 0x00;
	PORTC = 0x00;
	PORTA = 0xff;
	_delay_ms(1000);

}

// 숫자 출력 함수
void seg4_out(void)
{
	PORTA = (1<<DIG4)|(1<<DIGD1D2)|(1<<DIGAMPM);
	PORTF = number[N1];
	_delay_ms(1);

	PORTA = (1<<DIG3)|(1<<DIGD1D2)|(1<<DIGAMPM);
	PORTF = number[N10];
	_delay_ms(1);

	PORTA = (1<<DIG2)|(1<<DIGD1D2)|(1<<DIGAMPM);
	PORTF = number[N100];
	_delay_ms(1);

	PORTA = (1<<DIG1)|(1<<DIGD1D2)|(1<<DIGAMPM);
	PORTF = number[N1000];
	_delay_ms(1);
}

// 숫자 계산 함수
void number_count()
{
	// 1분 추가
	N1 = N1 + 1;
	
	// 10분이 된 경우 십의 자리 1추가, 일의 자리 0으로 초기화
	if(N1 == 10)
	{
		N1 = 0;
		N10 = N10 + 1;
	}
	
	// 60분이 된 경우 백의 자리 1 추가 및 십의 자리, 일의 자리 0으로 초기화
	if(N10 == 6 && N1 == 0)
	{
		N1 = 0; N10 = 0;
		N100 = N100 + 1;
	}
	
	// 10시간이 지난 경우 천의 자리 1 추가, 백의 자리 0으로 초기화
	if(N100 == 10)
	{
		N100 = 0;
		N1000 = N1000 + 1;
	}
	
	// 오후 12시가 된 경우 오전/오후 반전\
	// +)led 점등
	if(N1000 == 1 && N100 == 2 && N10 == 0 && N1 == 0 && AMPM ==0)
	{
		PORTC ^= (1<<AM)|(1<<PM); //XOR 연산
	}
	
	// 오후 13시가 되면 01시로 변경
	if(N1000 == 1 && N100 == 3 && AMPM == 0)
	{
		AMPM = 1;
		N1000 = 0;
		N100 = 1;
	}
	
	// 오전 12시가 된 경우 오전/오후 반전 및 00시로 변경
	// +)led 소등
	if(N1000 == 1 && N100 == 2 && AMPM == 1)
	{
		PORTB ^= (1<<AM)|(1<<PM); //XOR 연산
		AMPM = 0;
		N1000 = 0;
		N100 = 0;
	}
}

/*
int Display(void){

	PORTA = (1<<DIG4)|(1<<DIGD1D2)|(1<<DIGAMPM);
	PORTF = number[N1];
	_delay_ms(1);

	PORTA = (1<<DIG3)|(1<<DIGD1D2)|(1<<DIGAMPM);
	PORTF = number[N10];
	_delay_ms(1);

	PORTA = (1<<DIG2)|(1<<DIGD1D2)|(1<<DIGAMPM);
	PORTF = number[N100];
	_delay_ms(1);

	PORTA = (1<<DIG1)|(1<<DIGD1D2)|(1<<DIGAMPM);
	PORTF = number[N1000];
	_delay_ms(1);
}*/

void number_change()
{
	switch(pos)
	{
		case 0:
		N1 = N1 + 1;
		if(N1 == 10) N1 = 0;
		break;

		case 1:
		N10 = N10 + 1;
		if(N10==6) N10 = 0;
		break;

		case 2:
		N100 = N100 + 1;
		if(N100==3) N100 = 0;
		break;

		case 3:
		N1000 = N1000 + 1;
		if(N1000==2)
		{
			N1000 = 0;
			PORTB ^= (1<<AM)|(1<<PM);
		}
		break;
		default:
		pos = 0;
	}
}

int main(void)
{
	// DDR 지정
	DDRF = 0xff; // A, B, C, D, E, F, G
	DDRC = 0xff; // D1D2, AM, PM
	DDRA = 0xff; // DIG 트렌지스터
	PORTD=0X03;
	DDRD = 0x00; //스위치 부분
	
	cli(); // 인터럽트 정지
		
	// 타이머0 인터럽트 설정
	TCCR0 = (0<<CS00)|(1<<CS01)|(1<<CS02); // 프리스케일러 분주비 256 사용
	TCNT0 = 36;
	TIMSK = (1<<TOIE0); // 타이머0 오버플로우 인터럽트 활성화
	
   // INT0 인터럽트 설정
   EIMSK = (1<<INT0)|(1<<INT1);
   EICRA = (1<<ISC01)|(1<<ISC11);
   // SREG = 0x80; => sei() 가 동일한 동작을 함
   sei(); //인터럽트 시작
	
	init_seg(); // 초기화 함수 호출
	PORTC = (0<<AM)|(1<<PM); // AM/PM초기 설정(AM ON)
	
	while (1)
	{
		seg4_out();
		if(timer0_flag == 6)
		{
			number_count();
			timer0_flag = 0;
		}
	}
}

// INT0에서 인터럽트 발생시
ISR(INT0_vect, ISR_BLOCK)
{
	number_change();
}

// INT1에서 인터럽트 발생시
ISR(INT1_vect)
{
	pos++;
	
	if(pos==4)
		pos=0;
}

//타이머0에서 오버플로 인터럽트 발생시
ISR(TIMER0_OVF_vect,ISR_NOBLOCK)
{
	count++;
	if(count==60) //1초
	{
		PORTC ^= (1<<D1D2);
		timer0_flag++;
		count = 0;
	}
}

