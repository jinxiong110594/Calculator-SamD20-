///////////////////////////////////////////////////////////////////////////////////////////////////////////////////
// Code written by Justin Oroz and Jin Hua Xiong
///////////////////////////////////////////////////////////////////////////////////////////////////////////////////

//Include header files for all drivers
#include <asf.h>
#include <math.h>

// Enums and Structs
typedef enum key {NONE, ONE, TWO, THREE, ADD, FOUR, FIVE, SIX, SUBTRACT, SEVEN, EIGHT, NINE, MULTIPLY, DOT, ZERO, EQUALS, DIVIDE} key;

//Include void functions
void wait(float t);
void Simple_Clk_Init(void);
void initKeypad(void);
void initLED(void);
unsigned int getKeypadDigit(char);
void keyPressed(unsigned int keyVal);
void displayDigit(unsigned int digit);
void insertDigit(unsigned int digit);
void removeDigit();
unsigned int getKeyValue(key number);
int charToInt(char input);
char intToChar(int input);
void doMath(key function );
float pullValue(void);
void pushValue(float in);
void clearScreen(void);

//global variables
volatile char output[4] = {'\0', '\0', '\0', '\0'};
volatile bool decimal[4] = {false, false, false, false};
volatile bool negative = false;
volatile bool shift = false;


//sets the base address for the Port structure to PORT_INSTS or 0x41004400
Port *ports = PORT_INSTS;


int main (void)
{
	//set micro-controller clock to 8Mhz
	Simple_Clk_Init();
	initLED();
	initKeypad();


	//sets the group offset for the structure PortGroup in this case it is for group[0] or groupA
	// GroupA offset of 0x00				// GroupB offset of 0x80
	PortGroup *portA = &(ports->Group[0]);
	PortGroup *portB = &(ports->Group[1]);

	while (1){

		unsigned int pressedKey = 0;

		for (int i=0; i<4; i++) // rotate through rows
		{
			// turn on a single digit and row power
			portA->OUTSET.reg = PORT_PA07 | PORT_PA06 | PORT_PA05 | PORT_PA04;
			portA->OUTCLR.reg = PORT_PA07 >> i;

			//Set digit
			displayDigit(i);
			

			for (int j = 0; j<4; j++) // rotate through columns
			{
				// check if column pressed
				if (portA->IN.reg & (PORT_PA19 >> j)) {
					pressedKey = i * 4 + j + 1;
				}
			}
			
			// clears digit
			displayDigit(99);

		}
		
		keyPressed(pressedKey);
		pressedKey = 0;
	}

}

//LED Setup
void initLED(void) {

	PortGroup *portA = &(ports->Group[0]);
	PortGroup *portB = &(ports->Group[1]);

	// sets digits power to be outputs
	portA->DIRSET.reg = PORT_PA04 | PORT_PA05 | PORT_PA06 | PORT_PA07;

	// set digit segment switches to be outputs
	portB->DIRSET.reg = PORT_PB00 | PORT_PB01 | PORT_PB02 | PORT_PB03 | PORT_PB04 | PORT_PB05 | PORT_PB06 | PORT_PB07 | PORT_PB08 | PORT_PB09;

};

//Keypad Setup
void initKeypad(void){
	PortGroup *portA = &(ports->Group[0]);
	PortGroup *portB = &(ports->Group[1]);

	//sets row power to be output
	portA->DIRSET.reg = PORT_PA04 | PORT_PA05 | PORT_PA06 | PORT_PA07;

	// Set column ports 16-19 to be input
	portA->DIRCLR.reg = PORT_PA16 | PORT_PA17 | PORT_PA18 | PORT_PA19;

	// connects the Ports 16-19 to a pulldown
	portA->OUTCLR.reg = PORT_PA16 | PORT_PA17 | PORT_PA18 | PORT_PA19;
	//Perform other configurations for SW0 - Do input enable and pull-up

	for (int i=16; i<=19; i++)
	{
		portA->PINCFG[i].reg = PORT_PINCFG_INEN | PORT_PINCFG_PULLEN;
	}

}

int charToInt(char input) {
	//converts char to int
	if (input >= '0' && input <= '9') {
		return input - '0';
	} else {
		return 0;
	}
	
}

char intToChar(int input) {
	//converts int to char
	if (input >= 0 && input <= 9) {
		return '0' + input;
	} else {
		return '\0';
	}
	
}

void displayDigit(unsigned int digit) {
	//displays the proper character for the specified digit

	PortGroup *portB = &(ports->Group[1]);

	if (negative) { //if negative active LED
		portB->OUTCLR.reg = PORT_PA09;
	} else {
		portB->OUTSET.reg = PORT_PA09;
	}

	if (shift) { //if shifted activate segment
		portB->OUTCLR.reg = PORT_PB08;
	} else {
		portB->OUTSET.reg = PORT_PB08;
	}

	portB->OUTSET.reg = ~getKeypadDigit('\0');

	if (digit <4) {
		portB->OUTCLR.reg = getKeypadDigit(output[digit]);
		if (decimal[digit]) {
			portB->OUTCLR.reg = PORT_PB07;
		} else {
			portB->OUTSET.reg = PORT_PB07;
		}
	}
	
}


void keyPressed(unsigned int keyVal) {
	//responds to keypress inputs
	static key previousKey = NONE;
	static unsigned int pressedCount = 0;
	static unsigned int threshold = 20;
	static unsigned int clearThreshold = 2000;

	key pressedKey = (key) keyVal;

	if ((pressedKey == previousKey) && pressedKey != NONE) {
		// if a key is pressed and its the same as the previous scan
		pressedCount++;

		if (pressedCount == threshold) {
			//if the key has been pressed long enough

			switch (pressedKey) {
				case MULTIPLY:
				case DIVIDE:
				case ADD:
				case EQUALS:
					if (output[3] == '\0') {
						// if nothing on screen dont do another math function
						break;
					} else {
						doMath(pressedKey);
						break;
					}
					
					shift = false;
					break;
				case SUBTRACT:
					if (output[3] == '\0') {
						// if nothing on screen add negative sign
						//output[3] = '-';
						negative = !negative;
						break;
					} else {
						doMath(pressedKey);
						break;
					}
					
					shift = false;
					break;
				case DOT:
					// add dot
					// two decimals are impossible. dot twice clears
					if (decimal[0] || decimal[1] || decimal[2] || decimal[3]) {
						// change to backspace
						if (output[3] == '\0') {
							clearScreen();
							doMath(NONE); // clears math memory
						} else {
							removeDigit();
						}
					} else {
						decimal[3] = true;
					}
					break;
				default:
					// its a number
					// update value array
					insertDigit(getKeyValue(pressedKey));
			}
		} else if (pressedKey == DOT && pressedCount == clearThreshold) {
			//if decimal key is pressed long enough, clear screen
			clearScreen();
			doMath(NONE); // clears math memory
			shift = false;
		} else if (pressedKey == EQUALS && pressedCount == clearThreshold) {
			// toggle shift if equals held
			shift = !shift;
			//negative = shift;			//debugging
		}
	} else {
		pressedCount = 0;
		previousKey = pressedKey;
	}
}

float pullValue(void) {
	//pulls the value from the screen into an float
	float ans = 0;
	ans += 1000 * charToInt(output[0]);
	ans += 100 * charToInt(output[1]);
	ans += 10 *  charToInt(output[2]);
	ans +=  charToInt(output[3]);

	if (decimal[0]) {
		ans /= 1000;
	} else if (decimal[1]) {
		ans /= 100;
	} else if (decimal[2]) {
		ans /= 10;
	}


	if (negative) {
		ans *= -1;
	}

	return ans;
}

void pushValue(float in) {
	//pushes a float onto the display
	PortGroup *portB = &(ports->Group[1]);

	int val = (int) abs(in);

	if (in >= 9999.5 || in <= -9999.5) {
		output[0] = '\0';
		output[1] = '\0';
		output[2] = 'O';
		output[3] = 'F';
		return;
	} else if (abs(in) < 0.001 && in != 0) {
		output[0] = '\0';
		output[1] = '\0';
		output[2] = 'U';
		output[3] = 'F';
		return;
	}

	if (in < 0) {
		negative = true;
	} else {
		negative = false;
	}

	if (in == 0) {
			output[3] = '0';
			output[2] = '\0';
			output[1] = '\0';
			output[0] = '\0';
	} else if (abs(in) < 10) {
		if (fmod(in, 1) > 0) { // if floating point
			output[0] = intToChar((int) (val % 10));
			decimal[0] = true;

			int remainder = fmod(in, 1) *1000;
			output[1] = intToChar((int) remainder/100);
			output[2] = intToChar((int) (remainder % 100)/10);
			output[3] = intToChar((int) (remainder % 10));
		} else {
			output[3] = intToChar((int) (val % 10));
			output[2] = '\0';
			output[1] = '\0';
			output[0] = '\0';
		}	
	} else if (abs(in) < 100) { 
		if (fmod(in, 1) > 0) { // if floating point
			output[1] = intToChar((int) (val % 10));
			output[0] = intToChar((int) ((val % 100) / 10));
			decimal[1] = true;
			

			int remainder = fmod(in, 1) *1000;
			output[2] = intToChar((int) remainder/100);
			output[3] = intToChar((int) (remainder % 100)/10);
		} else {
			output[3] = intToChar((int) (val % 10));
			output[2] = intToChar((int) ((val % 100)/ 10));
			output[1] = '\0';
			output[0] = '\0';
		}
	} else if (abs(in) < 1000) {
		if (fmod(in, 1) > 0) { // if floating point
			output[2] = intToChar((int) (val % 10));
			output[0] = intToChar((int) ((val % 1000) / 100));
			output[1] = intToChar((int) ((val % 100)/ 10));
			decimal[2] = true;

			int remainder = fmod(in, 1) *1000;
			output[3] = intToChar((int) (remainder % 100)/10);
		} else {
			output[3] = intToChar((int) (val % 10));
			output[2] = intToChar((int) ((val % 100)/ 10));
			output[1] = intToChar((int) ((val % 1000) / 100));
			output[0] = '\0';
		}
	} else { // >= 1000
			output[3] = intToChar((int) (val % 10));
			output[2] = intToChar((int) ((val % 100)/ 10));
			output[1] = intToChar((int) ((val % 1000) / 100));
			output[0] = intToChar((int) (val / 1000));
	}

}

void doMath(key function) {
	//does the specified math function
	static float memory = 0;
	static key previousFunc = NONE;

	float current = pullValue();

	// commit previous math
	switch (previousFunc) {
		case MULTIPLY: //multiply
			memory *= current;
			break;
		case DIVIDE: //divide
			memory /= current;
			break;
		case ADD: //add
			memory += current;
			break;
		case SUBTRACT: // subtract
			memory -= current;
			break;
		case NONE:
		default:
			memory = current;
			break;
	}
	
	switch (function) {
		case MULTIPLY: //multiply
			if (output[3] == '\0') {
				// if nothing on screen dont do another math function
				return;
			} else {
				if (shift) {
					// shifted functions do not wait for second input, output answer
					clearScreen();
					memory *= memory;
					pushValue(memory);
					previousFunc = NONE;
			
				} else {
					previousFunc = function;
					clearScreen();
				}
				
			}
			break;
		case DIVIDE: //divide
			if (output[3] == '\0') {
					// if nothing on screen dont do another math function
					return;
				} else {
					if (shift) {
						// shifted functions do not wait for second input, output answer
						clearScreen();
						memory = sqrt(memory);
						pushValue(memory);
						previousFunc = NONE;
			
					} else {
						previousFunc = function;
						clearScreen();
					}
				
				}
				break;
		case ADD: //add
			if (output[3] == '\0') {
					// if nothing on screen dont do another math function
					return;
				} else {
					if (shift) {
						// shifted functions do not wait for second input, output answer
						clearScreen();
						memory = abs(memory);
						pushValue(memory);
						previousFunc = NONE;
			
					} else {
						previousFunc = function;
						clearScreen();
					}
				
				}
				break;
		case SUBTRACT: // subtract
			if (output[3] == '\0') {
				// if nothing on screen dont do another math function
				return;
			} else {
				if (shift) {
					// shifted functions do not wait for second input, output answer
					clearScreen();
					memory = log(memory);
					pushValue(memory);
					previousFunc = NONE;
			
				} else {
					previousFunc = function;
					clearScreen();
				}
				
			}
			break;
		case EQUALS: // Generate answer
			clearScreen();
			pushValue(memory);
			previousFunc = NONE;
			break;
		default:
			//resets memory, for clearing
			memory = 0;
			previousFunc = NONE;
	}
}

void clearScreen(void) {
	//makes screen blank
	for (int i=0; i< 4; i++) {
		output[i] = '\0';
		decimal[i] = false;
		negative = false;
	}
}

void insertDigit(unsigned int digit) {
	// inserts a digit into least significant position
	output[0] = output[1];
	output[1] = output[2];
	output[2] = output[3];
	output[3] = intToChar(digit);

	decimal[0] = decimal[1];
	decimal[1] = decimal[2];
	decimal[2] = decimal[3];
	decimal[3] = false;

}

void removeDigit() {
	//removes digit from least significant position
	output[3] = output[2];
	output[2] = output[1];
	output[1] = output[0];
	output[0] = '\0';

	decimal[3] = decimal[2];
	decimal[2] = decimal[1];
	decimal[1] = decimal[0];
	decimal[0] = false;

}

unsigned int getKeyValue(key number) {
	//returns integer value from key enumeration
	switch(number) {
		case ONE: 
			return 1;
		case TWO:
			return 2;
		case THREE: 
			return 3;
		case FOUR:
			return 4;
		case FIVE: 
			return 5;
		case SIX:
			return 6;
		case SEVEN: 
			return 7;
		case EIGHT:
			return 8;
		case NINE: 
			return 9;
		case ZERO:
		default:
			return 0;
	}
}

unsigned int getKeypadDigit(char input) {
	//returns ports required to create characters on displays
	switch(input) {

	case '0': //0
	case 'O': //Over
		return PORT_PB00 | PORT_PB01 | PORT_PB02 | PORT_PB03 | PORT_PB04 | PORT_PB05;
		break;
	case '1': //1
		return PORT_PB01 | PORT_PB02;
		break;
	case '2': //2
		return PORT_PB00 | PORT_PB01 | PORT_PB03 | PORT_PB04 | PORT_PB06;
		break;
	case '3': //3
		return PORT_PB00 | PORT_PB01 | PORT_PB02 | PORT_PB03 | PORT_PB06;
		break;
	case '4': //4
		return PORT_PB01 | PORT_PB02 | PORT_PB05 | PORT_PB06;
		break;
	case '5': //5
		return PORT_PB00 | PORT_PB02 | PORT_PB03 | PORT_PB05 | PORT_PB06;
		break;
	case '6': //6
		return PORT_PB00 | PORT_PB02 | PORT_PB03 | PORT_PB04 | PORT_PB05 | PORT_PB06;
		break;
	case '7': //7
		return PORT_PB00 | PORT_PB01 | PORT_PB02;
		break;
	case '8': //8
		return PORT_PB00 | PORT_PB01 | PORT_PB02 | PORT_PB03 | PORT_PB04 | PORT_PB05 | PORT_PB06;
		break;
	case '9': //9
		return PORT_PB00 | PORT_PB01 | PORT_PB02 | PORT_PB03 | PORT_PB05 | PORT_PB06;
		break;
	case 'U': //Under
		return PORT_PB01 | PORT_PB02 | PORT_PB03 | PORT_PB04 | PORT_PB05;
		break;
	case 'F': //Flow
		return PORT_PB00 | PORT_PB04 | PORT_PB05 | PORT_PB06;
		break;
	case '-': // negative sign
	return PORT_PB06;
		break;
	default: //clear
		return 0;
		break;
	}

}

//time delay function
void wait(float t)
{
	if (t < 0.001) {
		t = 0.001;
	}
	
	// is volatile needed?
	volatile int count = 0;
    while (count < t*1000)
	{
		count++;
	}
}

//Simple Clock Initialization - Do Not Modify -//
void Simple_Clk_Init(void)
{
	/* Various bits in the INTFLAG register can be set to one at startup.
	   This will ensure that these bits are cleared */
	
	SYSCTRL->INTFLAG.reg = SYSCTRL_INTFLAG_BOD33RDY | SYSCTRL_INTFLAG_BOD33DET |
			SYSCTRL_INTFLAG_DFLLRDY;
			
	system_flash_set_waitstates(0);  		//Clock_flash wait state =0

	SYSCTRL_OSC8M_Type temp = SYSCTRL->OSC8M;      	/* for OSC8M initialization  */

	temp.bit.PRESC    = 0;    			// no divide, i.e., set clock=8Mhz  (see page 170)
	temp.bit.ONDEMAND = 1;    			// On-demand is true
	temp.bit.RUNSTDBY = 0;    			// Standby is false
	
	SYSCTRL->OSC8M = temp;

	SYSCTRL->OSC8M.reg |= 0x1u << 1;  		// SYSCTRL_OSC8M_ENABLE bit = bit-1 (page 170)
	
	PM->CPUSEL.reg = (uint32_t)0;    		// CPU and BUS clocks Divide by 1  (see page 110)
	PM->APBASEL.reg = (uint32_t)0;     		// APBA clock 0= Divide by 1  (see page 110)
	PM->APBBSEL.reg = (uint32_t)0;     		// APBB clock 0= Divide by 1  (see page 110)
	PM->APBCSEL.reg = (uint32_t)0;     		// APBB clock 0= Divide by 1  (see page 110)

	PM->APBAMASK.reg |= 01u<<3;   			// Enable Generic clock controller clock (page 127)

	/* Software reset Generic clock to ensure it is re-initialized correctly */

	GCLK->CTRL.reg = 0x1u << 0;   			// Reset gen. clock (see page 94)
	while (GCLK->CTRL.reg & 0x1u ) {  /* Wait for reset to complete */ }
	
	// Initialization and enable generic clock #0

	*((uint8_t*)&GCLK->GENDIV.reg) = 0;  		// Select GCLK0 (page 104, Table 14-10)

	GCLK->GENDIV.reg  = 0x0100;   		 	// Divide by 1 for GCLK #0 (page 104)

	GCLK->GENCTRL.reg = 0x030600;  		 	// GCLK#0 enable, Source=6(OSC8M), IDC=1 (page 101)
}

///////////////////////////////////////////////////////////////////////////////////////////////////////////////
/* end of sample code for Lab 1 */
///////////////////////////////////////////////////////////////////////////////////////////////////////////////
