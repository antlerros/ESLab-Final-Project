# Hands-free Fan Control with Amazon Alexa Voice Service Using Raspberry Pi
以 Raspberry Pi 為媒介，利用聲控來操作小風扇。聲控的部分是使用 Amazon Alexa Voice Service (AVS)，並使用Alexa Skill Kit進行開發。 Alexa Skill 就是 Alexa 的擴充功能。

![](https://d2mxuefqeaa7sj.cloudfront.net/s_7F8784897FF7627480AE17DDC751000485EDD4F0E44531E0ACC4E9439C2F1431_1497439305735_photo_2017-06-14_19-11-35.jpg)

## **硬體設備**
- **Raspberry Pi 3** - 安裝 Alexa Voice Service 及 控制擴充功能的 Server。
- **Micro-USB 電源線** - Raspberry Pi 供電源。
- **USB 2.0 麥克風、小喇叭 (3.5mm jacks)** - 用來跟 Alexa 溝通。
- **L293 motor driving clip** - 類似風扇的開關，避免直接接觸 Raspberry Pi。
- **直流馬達 (6V)** - 風扇的軸心。
- **4號電池*4** - 馬達的供電源。
- **麵包板、導線、杜邦線** - 連接風扇跟 Raspberry Pi。
- **乙太網路線** - 用來將筆電跟 Raspberry Pi 連上，以便遠端操作。
## **使用方法**
1. 對著麥克風叫醒Alexa。
2. 聽到嗶聲之後講出指令。
3. Alexa 抓到關鍵字之後會啟動/關閉風扇，同時透過喇叭做出回應。
## **音訊溝通流程** 
1. 利用 [Sensory TrulyHandsFree](http://www.sensory.com/products/technologies/trulyhandsfree/) 可以啟動 Alexa。
2. 在 Alexa 啟動後會錄下指令，並上傳到 Amazon 的 Server。
3. Amazon 會利用 developer 設計的 interaction model 去 tokenize 上傳的指令，並把 tokenized 後的指令送到開發者指定的 service endpoint，執行對應的行為後會讓 Alexa 講出開發者設定的回覆。
4. 以下會針對我們自己開發的 Alexa Skill 做說明。
## **聲控風扇**
- **互動設計**
  - **Invocation name**
    必須先設定一個針對這個 skill 的 invocation name，這部分我是設定成 `**Raspberry Pi**`，所以當我叫醒 Alexa 之後，只要說 `tell` `**Raspberry Pi**` `to …` ，他就會啟動這個 skill 的 interaction model。
  - **Intent schema**
    接著必須設計 intent schema，有點類似設計 skill 的 interface，說明需要的 arguments (slots)。可以有不只一個 intent，而 intent schema 是用 JSON 來表示。
        {
          "intents": [
            {
              "intent": "GPIOControlIntent",
              "slots": [
                {
                  "name": "status",
                  "type": "GPIO_CONTROL"
                }
              ]
            }
          ]
        }
    slots 的 type 可以是 built-in 或是自行設定。在我們的例子我們的slot叫 status，而這個 slot 的 type 是自己設定的，所以需再另外寫 Custom Slots Type的 valid values（如下圖）。
https://d2mxuefqeaa7sj.cloudfront.net/s_7F8784897FF7627480AE17DDC751000485EDD4F0E44531E0ACC4E9439C2F1431_1497440930565_Screen+Shot+2017-06-14+at+19.47.51.png

  - 接著必須設計 sample utterances ，也就是使用者可能說的內容。以我們的例子會是 `**GPIOControlIntent**` `to turn the fan` `**{status}**`
    前面是 intent 的名字，後面則是 slot (argument) 的在這則語音指令裡的位置。
- **風扇控制**
  Fan control 的 code 是用 python 寫的。
  - **Web server**
    - 利用 Flask 及 flask_ask 這些 modules 在 Raspberry Pi 上架一個 web server，接收 requests 進而控制 Raspberry Pi 的 GPIO pins。intent 相當於 routing，而mapping則可以把先前指定的  argument 傳進來。回傳的 statement 則是開發者指定要 Alexa 在完成 request 後講的話。
    - 再利用 ngrok 來建立一個連接到 local web server 的 public url，並用這個 public url 當作前面提到的 service endpoint。
            from flask import Flask
            from flask_ask import Ask, statement, convert_errors
            
            @ask.intent('GPIOControlIntent', mapping={'status': 'status'})
            def gpio_control(status):
                
                """
                GPIO controll based on {status} ...
                """
                return statement('Turning fan {}'.format(status))
            if __name__ == '__main__':
                port = 5000 # the custom port you want, 5000 must be open/or port forwar$
                app.run(host='0.0.0.0', port=port)
  
  - **Circuit**
    為了控制馬達，我們購買了 L293D 馬達驅動晶片，主要是為了避免將 Raspberry Pi 直接接到馬達，而造成 Raspberry Pi 不必要的損害。因此，整個電路是以 L293D 為核心去串接而成，以下簡單介紹各個連接的pins以及該如何使馬達運作。
![](https://d2mxuefqeaa7sj.cloudfront.net/s_227F1E51FE01236FC3D832CB686A7A7280872961015871FF092C9B1C626C02FF_1497412154716_F5EFUKZINKX3299.MEDIUM.jpg)

    - Input 1 & Input 2
      L293D可以一次驅動兩顆馬達 (分成晶片左右半邊)，一顆馬達由兩個input去控制，而這次我們只用到了其中的 Input 1 和 Input 2。 Input 1 代表該馬達為順時鐘轉， Input 2 則表示馬達為逆時鐘旋轉。因此，在正常的情況之下，給定 Input 1 和 Input 2 的值為相反的。
    - Enable 1, 2 
      用來啟動馬達，因此將它維持在high。
    - Output 1 & Output 2
      根據 Input1, 2，將得到的 Output1, 2 分別接在馬達的兩端，來驅動馬達。
    - Vcc
      用來提供電源，這邊我們用了 4 顆 1.5V 的電池 (共6V) 來供電。不直接用 Raspberry Pi 來供電的原因是因為其所能供應的電流上限約為 20mA，然而要驅動馬達至少需要 400mA，因此我們選擇使用電池來供電。
  - **GPIO control**
    利用 RPi.GPIO 這個 module，在剛剛的 gpio_control(status) 裡面控制 GPIO pins。
    - 先在 global 設定 mode 來 access pins，這邊是用 Broadcom GPIO numbers (BCM)，並將 這些數字帶進變數中。
    - request 進來後設定這些 pins 為 output channels。根據 status argument 來設定 high, low。
          import RPi.GPIO as GPIO
          
          GPIO.setmode(GPIO.BCM)
          
          Motor1A = 02
          Motor1B = 03
          Motor1E = 04
          
          def gpio_control(status):
          
              GPIO.setup(Motor1A,GPIO.OUT)
              GPIO.setup(Motor1B,GPIO.OUT)
              GPIO.setup(Motor1E,GPIO.OUT)
          
              print "Motor going to Start"
              
              if status in ['on', 'high']:    
                  GPIO.output(Motor1A,GPIO.HIGH) 
                  GPIO.output(Motor1B,GPIO.LOW) 
                  GPIO.output(Motor1E,GPIO.HIGH) 
      
              if status in ['off', 'low']:    
                  GPIO.output(Motor1E,GPIO.LOW) # to stop the motor
          
              return statement('Turning fan {}'.format(status))
      

