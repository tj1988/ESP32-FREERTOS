# FreeRTOS 常用API

## vTaskDelay

```c++
void vTaskDelay( portTickType xTicksToDelay );
```

延时任务为已知时间片。任务被锁住剩余的实际时间由时间片率决定。portTICK_RATE_MS常量用来用来从时间片速率（一片周期代表着分辨率）来计算实际时间。

vTaskDelay()指定一个任务希望的时间段，这个时间之后（调用vTaskDelay() ）任务解锁。例如，指定周期段为100时间片，将使任务在调用vTaskDelay()100个时间片之后解锁。vTaskDelay()不提供一个控制周期性任务频率的好方法，像通过代码采取的路径，和其他任务和中断一样，在调用vTaskDelay()后 影响频率，因此任务所需的时间下一次执行。

```c++
vTaskDelay(  500 / portTICK_RATE_MS );//延迟500ms触发
```

## xTaskCreate

```c++
portBASE_TYPE xTaskCreate( 
   pdTASK_CODE pvTaskCode, 
   const portCHAR * const pcName, 
   unsigned portSHORT usStackDepth, 
   void *pvParameters, 
   unsigned portBASE_TYPE uxPriority, 
   xTaskHandle *pvCreatedTask 
 );
```

创建新的任务并添加到任务队列中，准备运行。

|   **参数**    |                           **用途**                           |
| :-----------: | :----------------------------------------------------------: |
|  pvTaskCode   | 指向任务的入口函数. 任务必须执行并且永不返回 (即：无限循环). |
|    pcName     | 描述任务的名字。主要便于调试。最大长度由configMAX_TASK_NAME_LEN.定义 |
| usStackDepth  | 指定任务堆栈的大小 ，堆栈能保护变量的数目- 不是字节数. 例如，如果堆栈为16位宽度，usStackDepth定义为 100, 200 字节，这些将分配给堆栈。堆栈嵌套深度（堆栈宽度）不能超多最大值——包含了size_t类型的变量 |
| pvParameters  |              指针用于作为一个参数传向创建的任务              |
|  uxPriority   |                      任务运行时的优先级                      |
| pvCreatedTask |               用于传递一个处理——引用创建的任务               |

使用例子：

```c++
// 创建任务
void vTaskCode( void * pvParameters ){
  for( ;; ){
    // 任务代码
  }
}
// 函数来创建一个任务
void vOtherFunction( void ){
  static unsigned char ucParameterToPass;
  xTaskHandle xHandle;
  // 创建任务，存储处理。注意传递的参数为ucParameterToPass
  // 它在任务中不能始终存在, 所以定义为静态变量. 如果它是动态堆栈的变量，可能存在
  // 没有那么长，或者至少随着时间毁灭，
  // 新的时间， 尝试存储它
  // 参数：任务函数，任务别名，任务堆栈的深度，参数的指针，任务优先级，回传句柄
  xTaskCreate( vTaskCode, "NAME", STACK_SIZE, &ucParameterToPass, tskIDLE_PRIORITY, &xHandle );
  // 使用句柄来删除任务
  vTaskDelete( xHandle );
}
```

使用例子2：（Arduino）

```c++
#include "freertos/FreeRTOS.h"
TaskHandle_t task_handle1; //普通任务
TaskHandle_t task_handle2; //指定运行内核任务
void task1(void *param)
{
  while (1)
  {
    
   }
    vTaskDelay(10);
}
void task2(void *param)
{
  while (1)
  {
    
   }
    vTaskDelay(10);
}
void setup(){
    xTaskCreate(task1, "task1", 1024 * 2, NULL, 3, &task_handle1);//创建普通任务
    xTaskCreatePinnedToCore(
      task2,
      "task2",
      1024 * 8,
      NULL,
      4,
      &task_handle2,
      1); //创建指定内核任务
}
void loop(){
     vTaskDelay(pdMS_TO_TICKS(1000));
}
```



## xQueueCreate

```c++
xQueueHandle xQueueCreate( 
   unsigned portBASE_TYPE uxQueueLength, /*消息个数 */
   unsigned portBASE_TYPE uxItemSize  /* 每个消息大小，单位字节 */
);
```

创建一个新的消息队列。为新的队列分配所需的存储内存，并返回一个队列处理。

**返回值**:如果创建成功会返回消息队列的句柄，如果由于 FreeRTOSConfig.h 文件中 heap 大小不足，无法为此消息队列提供所需的空间会返回 NULL。

使用例子：

```c++
struct AMessage {
    portCHAR ucMessageID;
    portCHAR ucData[ 20 ];
};
void vATask( void *pvParameters ){
    xQueueHandle xQueue1, xQueue2;
    // 创建一个队列，包含10个unsigned long值
    xQueue1 = xQueueCreate( 10, sizeof( unsigned portLONG ) );
    if( xQueue1 == 0 ){
        // 队列不能创建，就不能使用
    }
    // 创建一个队列，包含10个指向AMessage 结构的指针
    /// 可以通过指针传递，指针可以包含很多数据
    xQueue2 = xQueueCreate( 10, sizeof( struct AMessage * ) );
    if( xQueue2 == 0 ){
        // 队列不能创建，就不能使用
    }
    // ... 其余代码
}
```

## xQueueSend

```c++
portBASE_TYPE xQueueSend( 
   xQueueHandle xQueue, /* 消息队列句柄 */
   const void * pvItemToQueue, /* 要传递数据地址 */
   portTickType xTicksToWait /* 等待消息队列有空间的最大等待时间 */
);
```

1. FreeRTOS 的消息传递是数据的复制，而不是传递的数据地址。
2. 此函数是用于任务代码中调用的，故不可以在中断服务程序中调用此函数，中断服务程序中使用的是xQueueSendFromISR。
3. 如果消息队列已经满且第三个参数为 0，那么此函数会立即返回。
4. **返回值**，如果消息成功发送返回 pdTRUE，否则返回 errQUEUE_FULL。

使用范例：

```c++
struct AMessage
 {
    portCHAR ucMessageID;
    portCHAR ucData[ 20 ];
 } xMessage;
 
 unsigned portLONG ulVar = 10UL;
 
 void vATask( void *pvParameters )
 {
 xQueueHandle xQueue1, xQueue2;
 struct AMessage *pxMessage;
  // 创建一个队列，包含10个unsigned long值
  xQueue1 = xQueueCreate( 10, sizeof( unsigned portLONG ) );
  // 创建一个队列，包含10个指针指向AMessage 结构的指针
  xQueue2 = xQueueCreate( 10, sizeof( struct AMessage * ) );
  // 可以通过指针传递，指针可以包含很多数据

  if( xQueue1 != 0 )
   {
   // 传递一个unsigned long。等待10个时间片，分配所需的可用空间 
   if( xQueueSend( xQueue1, ( void * ) &ulVar, ( portTickType ) 10 ) != pdPASS )
    {
         // 传递信息失败，继续等待下一个10个时间片。
    }
   }
     
  if( xQueue2 != 0 )
    {
        // 传递一个指向 AMessage 结构的指针。如果队列已经满，不要锁住
        pxMessage = & xMessage;
        xQueueSend( xQueue2, ( void * ) &pxMessage, ( portTickType ) 0 );
    } 
     	// ... 其余代码
     
 }
```

## xQueueSendFromISR

```c++
portBASE_TYPE xQueueSendFromISR(
  xQueueHandle pxQueue,/* 消息队列句柄 */
  const void *pvItemToQueue,/* 要传递数据地址 */
  portBASE_TYPE *pxHigherPriorityTaskWoken/* 高优先级任务是否被唤醒的状态保存 */
);
```

第3个参数用于保存是否有高优先级任务准备就绪。如果函数执行完毕后，此参数的数值是pdTRUE，说明有高优先级任务要执行，否则没有。

返回值，如果消息成功发送返回 pdTRUE，否则返回 errQUEUE_FULL

消息队列还有两个函数 xQueueSendToBackFromISR 和 xQueueSendToFrontFromISR，

函数xQueueSendToBackFromISR 实现的是 FIFO 方式的存取，函数 xQueueSendToFrontFromISR 实现的是 LIFO 方式的读写。

我们这里说的函数 xQueueSendFromISR 等效于xQueueSendToBackFromISR，即实现的是 FIFO 方式的存取。

使用范例：

```c++
static bool timer_callback(void *args){
    uint64_t val;
    BaseType_t pxHigherPriorityTaskWoken = pdFALSE;
    
    val = timer_group_get_counter_value_in_isr(0, 0);
    /*
     * 上行代码：
     * ————————————————————
     * 将定时器的值传给一个任务
     *（由于本示例使用的自动重装载模式，
     * 所以在本示例中这个val值无意义。
     * 只是为了展示在isr callback中获
     * 取定时器值函数的使用【必须
     * 调用带有_in_isr的函数】）
     * ————————————————————
     */
    
    //通过队列将 val 传给任务
    xQueueSendFromISR(queue, (void*)&val, &pxHigherPriorityTaskWoken);

    return pxHigherPriorityTaskWoken;
}
```

## xQueueReceive

```c++
portBASE_TYPE xQueueReceive( 
  xQueueHandle xQueue, /* 消息队列句柄 */
  void *pvBuffer, /* 接收消息队列数据的缓冲地址 */
  portTickType xTicksToWait /* 等待消息队列有数据的最大等待时间 */
);
```

第 2 个参数是从消息队列中复制出数据后所储存的缓冲地址，缓冲区空间要大于等于消息队列创建函数 xQueueCreate 所指定的单个消息大小，否则取出的数据无法全部存储到缓冲区，从而造成内存溢出。

返回值，如果接收到消息返回 pdTRUE，否则返回 pdFALSE

```c++
 uint8_t ucQueueMsgValue;                    
        xResult = xQueueReceive(xQueue1,                   /* 消息队列句柄 */
                                (void *)&ucQueueMsgValue,  /* 存储接收到的数据到变量ucQueueMsgValue中 */
                                (TickType_t)xMaxBlockTime);/* 设置阻塞时间 */
        
        if(xResult == pdPASS)
        {
            /* 成功接收，并通过串口将数据打印出来 */
            printf("接收到消息队列数据ucQueueMsgValue = %d\r\n", ucQueueMsgValue);
        }
```

```c++
  MSG_T *ptMsg;
        xResult = xQueueReceive(xQueue2,                   /* 消息队列句柄 */
                                (void *)&ptMsg,             /* 这里获取的是结构体的地址 */
                                (TickType_t)xMaxBlockTime);/* 设置阻塞时间 */
        
        
        if(xResult == pdPASS)
        {
            /* 成功接收，并通过串口将数据打印出来 */
            printf("接收到消息队列数据ptMsg->ucMessageID = %d\r\n", ptMsg->ucMessageID);
            printf("接收到消息队列数据ptMsg->ulData[0] = %d\r\n", ptMsg->ulData[0]);
            printf("接收到消息队列数据ptMsg->usData[0] = %d\r\n", ptMsg->usData[0]);
        }
```

