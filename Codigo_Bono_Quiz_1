#include <stdio.h>
#include "freertos/FreeRTOS.h"
#include "freertos/task.h"
#include "driver/gpio.h"
#include "driver/ledc.h"
#include "driver/adc.h"
#include "esp_timer.h"


#define LED_GREEN 2
#define LED_RED   5

#define BTN_RIGHT 0
#define BTN_LEFT  22

#define DIR_A 19
#define DIR_B 21

#define PWM_NPN 18   // 2N2222A
#define PWM_PNP 23   // 2N3906

#define POT ADC1_CHANNEL_6

int seg_pins[7]   = {12,13,14,15,25,26,27};
int digit_pins[3] = {4,32,33};


volatile int current_digit = 0;
volatile int display_number = 0;
volatile int pwm_value = 0;


const uint8_t numbers[10][7] = {
    {1,1,1,1,1,1,0},{0,1,1,0,0,0,0},{1,1,0,1,1,0,1},
    {1,1,1,1,0,0,1},{0,1,1,0,0,1,1},{1,0,1,1,0,1,1},
    {1,0,1,1,1,1,1},{1,1,1,0,0,0,0},{1,1,1,1,1,1,1},
    {1,1,1,1,0,1,1}
};


void multiplex_display(void* arg)
{
    for(int i=0;i<3;i++)
        gpio_set_level(digit_pins[i], 0);

    int digit;

    if(current_digit == 0) digit = display_number % 10;
    else if(current_digit == 1) digit = (display_number/10)%10;
    else digit = (display_number/100)%10;

    for(int i=0;i<7;i++)
        gpio_set_level(seg_pins[i], !numbers[digit][i]);

    gpio_set_level(digit_pins[current_digit], 1);

    current_digit = (current_digit + 1) % 3;
}


void pwm_control(void* arg)
{
    static int counter = 0;

    counter++;
    if(counter >= 255) counter = 0;

    // NPN (directo)
    gpio_set_level(PWM_NPN, (counter < pwm_value));

    // PNP (invertido)
    gpio_set_level(PWM_PNP, !(counter < pwm_value));
}


void app_main(void)
{
    // GPIO OUTPUTS
    int outputs[] = {LED_GREEN, LED_RED, DIR_A, DIR_B, PWM_NPN, PWM_PNP};

    for(int i=0;i<6;i++){
        gpio_reset_pin(outputs[i]);
        gpio_set_direction(outputs[i], GPIO_MODE_OUTPUT);
    }

    // Segmentos
    for(int i=0;i<7;i++){
        gpio_reset_pin(seg_pins[i]);
        gpio_set_direction(seg_pins[i], GPIO_MODE_OUTPUT);
    }

    for(int i=0;i<3;i++){
        gpio_reset_pin(digit_pins[i]);
        gpio_set_direction(digit_pins[i], GPIO_MODE_OUTPUT);
    }

    // Botones
    gpio_set_direction(BTN_RIGHT, GPIO_MODE_INPUT);
    gpio_set_pull_mode(BTN_RIGHT, GPIO_PULLUP_ONLY);

    gpio_set_direction(BTN_LEFT, GPIO_MODE_INPUT);
    gpio_set_pull_mode(BTN_LEFT, GPIO_PULLUP_ONLY);

    // ADC
    adc1_config_width(ADC_WIDTH_BIT_12);
    adc1_config_channel_atten(POT, ADC_ATTEN_DB_12);

    // TIMER DISPLAY
    esp_timer_handle_t display_timer;
    esp_timer_create_args_t display_args = {
        .callback = &multiplex_display
    };
    esp_timer_create(&display_args, &display_timer);
    esp_timer_start_periodic(display_timer, 2000);

    // TIMER PWM 
    esp_timer_handle_t pwm_timer;
    esp_timer_create_args_t pwm_args = {
        .callback = &pwm_control
    };
    esp_timer_create(&pwm_args, &pwm_timer);

    // 50 µs → ~20 kHz base loop → PWM efectivo ~80 Hz
    esp_timer_start_periodic(pwm_timer, 50);

    // PRINCIPAL
    while(1)
    {
        int adc = adc1_get_raw(POT);
        pwm_value = adc / 16;

        int btn_r = !gpio_get_level(BTN_RIGHT);
        int btn_l = !gpio_get_level(BTN_LEFT);

        // Cambio de dirección del motor y leds
        if(btn_r){
            gpio_set_level(DIR_A, 1);
            gpio_set_level(DIR_B, 0);

            gpio_set_level(LED_GREEN, 1);
            gpio_set_level(LED_RED, 0);
        }
        else if(btn_l){
            gpio_set_level(DIR_A, 0);
            gpio_set_level(DIR_B, 1);

            gpio_set_level(LED_GREEN, 0);
            gpio_set_level(LED_RED, 1);
        }

        display_number = abs(((pwm_value * 100) / 255) - 100);

        printf("ADC:%d PWM:%d BTN_R:%d BTN_L:%d\n",
               adc, pwm_value, btn_r, btn_l);

        vTaskDelay(pdMS_TO_TICKS(100));
    }
}
