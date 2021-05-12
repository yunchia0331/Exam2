# Exam2
*array存特性分析的值
int array[10];
int feature=1;

void print_gesture_ID() {
    uLCD.cls();
    uLCD.color(BLACK);
    uLCD.locate(1, 2);
    uLCD.printf("\n0\n");    
    uLCD.locate(1, 4);
    uLCD.printf("\n1\n");
    uLCD.locate(1, 6);
    uLCD.printf("\n2\n");
    
*如果加速器測出第一種圖(snoe==0)，設0，印在ULCD
    if (snoe==0) {
        uLCD.color(RED);
        uLCD.locate(1, 2);
        uLCD.printf("\n0\n");
    } else if (snoe==1) {
        uLCD.color(RED);
        uLCD.locate(1, 4);
        uLCD.printf("\n1\n");
    } else if (snoe==2) {
        uLCD.color(RED);
        uLCD.locate(1, 6);
        uLCD.printf("\n2\n");
    } 
}
void print_angle_detect() {
    uLCD.cls();
    uLCD.color(BLACK);
    uLCD.locate(1, 2);
    uLCD.printf("\n%.2f degree\n", angle_det);    
}
/*********************************************
/*BELOW: RPC and thread function*/
/*************************************************************************************/
/*************************************************************************************/
void Leave_Mode(Arguments *in, Reply *out) {
    printf("\r\nSuccessfully leave the mode\r\n");
    num = 0;

}


void Acc(Arguments *in, Reply *out) {          //In this accelerator capture mode, mbed will start a thread function.

    //id = gesture_queue.cal(acc);
    gesture_thread.start(callback(&gesture_queue, &EventQueue::dispatch_forever));
    gesture_queue.call(acc);
}


void acc() {
    
    while (num==1) {                                                                // num 1: Acc mode

> mbed will record a set of accelerator data values
        gesture_index = PredictGesture(interpreter->output(0)->data.f); 

        // Clear the buffer next time we read data
        should_clear_buffer = gesture_index < label_num;

        // Produce an output    gesture ID and sequence number of the event) through WiFi/MQTT to a broker
        if (gesture_index < label_num) {
            error_reporter->Report(config.output_message[gesture_index]);
 > 如果加速器測出第一種圖(snoe==0)，設0，印在ULCD
 
            if (gesture_index == 0) {                                   //  sequence number of the event
                snoe = 0;         
                queue.call(print_gesture_ID);
                mqtt_queue.call(&publish_message, &client);
            }
            else if (gesture_index == 1) {
                snoe = 1;
                queue.call(print_gesture_ID);
                mqtt_queue.call(&publish_message, &client);
            }
            else if (gesture_index == 2) {
                snoe = 2;
                queue.call(print_gesture_ID);
                mqtt_queue.call(&publish_message, &client);
            }

        }
    }
}



void Angle_Detection(Arguments *in, Reply *out) {

    angle_detection_thread.start(callback(&angle_detection_queue, &EventQueue::dispatch_forever));
    angle_detection_queue.call(angle_detection);
}


void angle_detection() {
    //for Accelerometer
    int16_t pDataXYZ_init[3] = {0};
    int16_t pDataXYZ[3] = {0};
    double mag_A;
    double mag_B;
    double cos;
    double rad_det;
 
    printf("enter angle detection mode\r\n");
    printf("Place the mbed on table after LEDs\r\n");
    ThisThread::sleep_for(2000ms);

    
    BSP_ACCELERO_AccGetXYZ(pDataXYZ_init);
    printf("reference acceleration vector: %d, %d, %d\r\n", pDataXYZ_init[0], pDataXYZ_init[1], pDataXYZ_init[2]);

    printf("Tilt the mbed \r\n");
    ThisThread::sleep_for(2000ms);
    
    num = 2;                                    // tilt Angle_Detection mode
    while (num==2) {
        for(a=0 ;a<10;a++){
        BSP_ACCELERO_AccGetXYZ(pDataXYZ);
        printf("Angle Detection: %d %d %d\r\n",pDataXYZ[0], pDataXYZ[1], pDataXYZ[2]);
        mag_A = sqrt(pDataXYZ_init[0]*pDataXYZ_init[0] + pDataXYZ_init[1]*pDataXYZ_init[1] + pDataXYZ_init[2]*pDataXYZ_init[2]);
        mag_B = sqrt(pDataXYZ[0]*pDataXYZ[0] + pDataXYZ[1]*pDataXYZ[1] + pDataXYZ[2]*pDataXYZ[2]);
        cos = ((pDataXYZ_init[0]*pDataXYZ[0] + pDataXYZ_init[1]*pDataXYZ[1] + pDataXYZ_init[2]*pDataXYZ[2])/(mag_A)/(mag_B));
        rad_det = acos(cos);
        angle_det = 180.0 * rad_det/PI;
        printf("angle_det = %.2f\r\n", angle_det);
        queue.call(print_angle_detect);
                                                //做特性分析
        
            if (angle_det > 30) {
                printf("change of direction");
                mqtt_queue.call(&publish_message, &client);
                feature = 1;
            }else{
                printf("stationary");
                feature = 0;
            }
            array[a]=feature;
            printf("%d\r\n",array[a]);
        }
        ThisThread::sleep_for(1000ms);
        break;
    }
}
/*************************************************************************************/
/*************************************************************************************/
/*Above: RPC and thread function*/



int main() {
    uLCD.background_color(WHITE);                                                           // initial display on uLCD
    uLCD.cls();
    uLCD.textbackground_color(WHITE);
    uLCD.color(RED);
    uLCD.locate(1, 2);
    uLCD.printf("\n15\n");
    uLCD.color(BLACK);
    uLCD.locate(1, 4);
    uLCD.printf("\n45\n");
    uLCD.locate(1, 6);
    uLCD.printf("\n60\n");


    thread.start(callback(&queue, &EventQueue::dispatch_forever));                          // for output on uLCD
    mqtt_thread.start(callback(&mqtt_queue, &EventQueue::dispatch_forever));

    /*BELOW: MQTT*/
    wifi = WiFiInterface::get_default_instance();
    if (!wifi) {
            printf("ERROR: No WiFiInterface found.\r\n");
            //return -1;
    }

    printf("\nConnecting to %s...\r\n", MBED_CONF_APP_WIFI_SSID);
    int ret = wifi->connect(MBED_CONF_APP_WIFI_SSID, MBED_CONF_APP_WIFI_PASSWORD, NSAPI_SECURITY_WPA_WPA2);
    if (ret != 0) {
            printf("\nConnection error: %d\r\n", ret);
            //return -1;
    }
    
    NetworkInterface* net = wifi;
    MQTTNetwork mqttNetwork(net);
    MQTT::Client<MQTTNetwork, Countdown> client(mqttNetwork);


    //TODO: revise host to your IP
    const char* host = "192.168.3.118";
    printf("Connecting to TCP network...\r\n");

    SocketAddress sockAddr;
    sockAddr.set_ip_address(host);
    sockAddr.set_port(1883);

    printf("address is %s/%d\r\n", (sockAddr.get_ip_address() ? sockAddr.get_ip_address() : "None"),  (sockAddr.get_port() ? sockAddr.get_port() : 0) ); //check setting

    int rc = mqttNetwork.connect(sockAddr);//(host, 1883);
    if (rc != 0) {
            printf("Connection error.");
            //return -1;
    }
    printf("Successfully connected!\r\n");

    MQTTPacket_connectData data = MQTTPacket_connectData_initializer;
    data.MQTTVersion = 3;
    data.clientID.cstring = "Mbed";

    if ((rc = client.connect(data)) != 0){
            printf("Fail to connect MQTT\r\n");
    }
    if (client.subscribe(topic, MQTT::QOS0, messageArrived) != 0){
            printf("Fail to subscribe\r\n");
    }    
  /*Above: MQTT*/

    printf("Start accelerometer init\n");
    BSP_ACCELERO_Init();
// btn.rise(mqtt_queue.event(&publish_message, &client));


    //The mbed RPC classes are now wrapped to create an RPC enabled version - see RpcClasses.h so don't add to base class
    // receive commands, and send back the responses
    char buf[256], outbuf[256];

    FILE *devin = fdopen(&pc, "r");
    FILE *devout = fdopen(&pc, "w");

    while(1) {
        memset(buf, 0, 256);
        for (int i = 0; ; i++) {
            char recv = fgetc(devin);
            if (recv == '\n') {
                printf("\r\n");
                break;
            }
            buf[i] = fputc(recv, devout);
        }
        //Call the static call method on the RPC class
        RPC::call(buf, outbuf);
        printf("%s\r\n", outbuf);
    }



    int num = 0;
    while (num != 5) {
            client.yield(100);
            ++num;
    }

    while (1) {
            if (closed) break;
            client.yield(500);
            ThisThread::sleep_for(500ms);
    }
/*
    printf("Ready to close MQTT Network......\n");

    if ((rc = client1.unsubscribe(topic1)) != 0) {
            printf("Failed: rc from unsubscribe was %d\n", rc);
    }

    if ((rc = client1.disconnect()) != 0) {
    printf("Failed: rc from disconnect was %d\n", rc);
    }

    mqttNetwork.disconnect();
    printf("Successfully closed!\n");
*/
}
