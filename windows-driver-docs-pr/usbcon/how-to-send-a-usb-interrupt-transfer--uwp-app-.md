---
Description: A USB device can support interrupt endpoints so that it can send or receive data at regular intervals.
title: How to send a USB interrupt transfer request (Windows Store app)
author: windows-driver-content
ms.author: windowsdriverdev
ms.date: 04/20/2017
ms.topic: article
ms.prod: windows-hardware
ms.technology: windows-devices
---

# How to send a USB interrupt transfer request (Windows Store app)


**Summary**

-   Interrupt transfers occur when the host polls the device
-   Implement the event handler for [**UsbInterruptInPipe.DataReceived**](https://msdn.microsoft.com/library/windows/apps/dn278418)
-   Register and unregister the event handler

**Important APIs**

-   [**UsbInterruptInPipe**](https://msdn.microsoft.com/library/windows/apps/dn278416)
-   [**UsbInterruptOutPipe**](https://msdn.microsoft.com/library/windows/apps/dn278425)
-   [**UsbInterruptInEventArgs**](https://msdn.microsoft.com/library/windows/apps/dn278414)

A USB device can support interrupt endpoints so that it can send or receive data at regular intervals. To accomplish that, the host polls the device at regular intervals and data is transmitted each time the host polls the device. Interrupt transfers are mostly used for getting interrupt data from the device. This topic describes how a Windows Store app can get continuous interrupt data from the device.

**Interrupt endpoint information**

For interrupt endpoints, the descriptor exposes these properties. Those values are for information only and should not affect how you manage the buffer transfer buffer.

-   How often can data be transmitted?

    Get that information by getting the **Interval** value of the endpoint descriptor (see [**UsbInterruptOutEndpointDescriptor.Interval**](https://msdn.microsoft.com/library/windows/apps/dn278422) or [**UsbInterruptInEndpointDescriptor.Interval**](https://msdn.microsoft.com/library/windows/apps/dn278411)). That value indicates how often data is sent to or received from the device in each frame on the bus.

    **Note**  The **Interval** property is not the **bInterval** value (defined in the USB specification).

     

    That value indicates how often data is transmitted to or from the device. For example, for a high speed device, if **Interval** is 125 microseconds, data is transmitted every 125 microseconds. If **Interval** is 1000 microseconds, then data is transmitted every millisecond.

-   How much data can be transmitted in each service interval?

    Get the number of bytes that can transmitted by getting the maximum packet size supported by the endpoint descriptor (see UsbInterruptOutEndpointDescriptor.MaxPacketSize or UsbInterruptInEndpointDescriptor.MaxPacketSize. The maximum packet size constrained on the speed of the device. For low-speed devices up to 8 bytes. For full-speed devices, up to 64 bytes. For high-speed, high-bandwidth devices, the app can send or receive more than maximum packet size up to 3072 bytes per microframe.

    **Note**  Interrupt endpoints on SuperSpeed devices are capable of transmitting even more number of bytes. That value is indicated by the **wBytesPerInterval** of the USB\_SUPERSPEED\_ENDPOINT\_COMPANION\_DESCRIPTOR. To retrieve the descriptor, get the descriptor buffer by using the [**UsbEndpointDescriptor.AsByte**](https://msdn.microsoft.com/library/windows/apps/dn263827) property and then parse that buffer by using [**DataReader**](https://msdn.microsoft.com/library/windows/apps/br208119) methods.

     

**Interrupt OUT transfers**

A USB device can support interrupt OUT endpoints that receive data from the host at regular intervals. Each time the host polls the device, the host sends data. A Windows Store app can initiate an interrupt OUT transfer request that specifies the data to send. That request is completed when the device acknowledges the data from the host. A Windows Store app can write data to the [**UsbInterruptOutPipe**](https://msdn.microsoft.com/library/windows/apps/dn278425).

**Interrupt IN transfers**

Conversely, a USB device can support interrupt IN endpoints as a way to inform the host about hardware interrupts generated by the device. Typically USB Human Interface Devices (HID) such as keyboards and pointing devices support interrupt OUT endpoints. When an interrupt occurs, the endpoint stores interrupt data but that data does not reach the host immediately. The endpoint must wait for the host controller to poll the device. Because there must be minimal delay between the time data is generated and reaches the host, it polls the device at regular intervals. A Windows Store app can get data received in the [**UsbInterruptInPipe**](https://msdn.microsoft.com/library/windows/apps/dn278416). The request that completes when data from the device is received by the host.

## Before you start...


-   You must have opened the device and obtained the [**UsbDevice**](https://msdn.microsoft.com/library/windows/apps/dn263883) object. Read [How to connect to a USB device (Windows Store App)](how-to-connect-to-a-usb-device--windows-store-app-.md).
-   You can see the complete code shown in this topic in the CustomUsbDeviceAccess sample, Scenario3\_InterruptPipes files.

## Writing to the interrupt OUT endpoint


The way the app sends an interrupt OUT transfer request is identical to bulk OUT transfers, except the target is an interrupt OUT pipe, represented by [**UsbInterruptOutPipe**](https://msdn.microsoft.com/library/windows/apps/dn278425). For more information, see [How to send a USB bulk transfer request (Windows Store app)](how-to-send-a-usb-bulk-transfer--windows-store-app-.md).

## Step 1: Implement the interrupt event handler (Interrupt IN)


When data is received from the device into the interrupt pipe, it raises the [**DataReceived**](https://msdn.microsoft.com/library/windows/apps/dn278418) event. To get the interrupt data, the app must implement an event handler. The *eventArgs* parameter of the handler, points to the data buffer.

This example code shows a simple implementation of the event handler. The handler maintains the count of interrupts received. Each time the handler is invoked, it increments the count. The handler gets the data buffer from *eventArgs* parameter and displays the count of interrupts and the length of bytes received.

```CSharp
private async void OnInterruptDataReceivedEvent(UsbInterruptInPipe sender, UsbInterruptInEventArgs eventArgs)
{
    numInterruptsReceived++;

    // The data from the interrupt
    IBuffer buffer = eventArgs.InterruptData;

    // Create a DispatchedHandler for the because we are interracting with the UI directly and the
    // thread that this function is running on may not be the UI thread; if a non-UI thread modifies
    // the UI, an exception is thrown

    await Dispatcher.RunAsync(
                       CoreDispatcherPriority.Normal,
                       new DispatchedHandler(() =>
    {
        ShowData(
        "Number of interrupt events received: " + numInterruptsReceived.ToString()
        + "\nReceived " + buffer.Length.ToString() + " bytes");
    }));
}
```

```ManagedCPlusPlus
void OnInterruptDataReceivedEvent(UsbInterruptInPipe^ /* sender */, UsbInterruptInEventArgs^  eventArgs )
{
    numInterruptsReceived++;

    // The data from the interrupt
    IBuffer^ buffer = eventArgs->InterruptData;

    // Create a DispatchedHandler for the because we are interracting with the UI directly and the
    // thread that this function is running on may not be the UI thread; if a non-UI thread modifies
    // the UI, an exception is thrown

    MainPage::Current->Dispatcher->RunAsync(
        CoreDispatcherPriority::Normal,
        ref new DispatchedHandler([this, buffer]()
        {
            ShowData(
                "Number of interrupt events received: " + numInterruptsReceived.ToString()
                + "\nReceived " + buffer->Length.ToString() + " bytes",
                NotifyType::StatusMessage);
        }));
}
```

## Step 2: Get the interrupt pipe object (Interrupt IN)


To register the event handler for the [**DataReceived**](https://msdn.microsoft.com/library/windows/apps/dn278418) event, obtain a reference to the [**UsbInterruptInPipe**](https://msdn.microsoft.com/library/windows/apps/dn278416) by using any these properties:

-   [**UsbDevice.DefaultInterface.InterruptInPipes\[n\]**](https://msdn.microsoft.com/library/windows/apps/dn264292) if your interrupt endpoint is present in the first USB interface.
-   [**UsbDevice.Configuration.UsbInterfaces\[m\].InterruptInPipes\[n\]**](https://msdn.microsoft.com/library/windows/apps/dn264292) for enumerating all interrupt IN pipes per interface, supported by the device.
-   [**UsbInterface.InterfaceSettings\[m\].InterruptInEndpoints \[n\].Pipe**](https://msdn.microsoft.com/library/windows/apps/dn264292) for enumerating interrupt IN pipes defined by settings of an interface.
-   [**UsbEndpointDescriptor.AsInterruptInEndpointDescriptor.Pipe**](https://msdn.microsoft.com/library/windows/apps/dn278413) for getting the pipe object from the endpoint descriptor for the interrupt IN endpoint.

**Note**  Avoid getting the pipe object by enumerating interrupt endpoints of an interface setting that is not currently selected. To transfer data, pipes must be associated with endpoints in the active setting.

 

## Step 3: Register the event handler to start receiving data (Interrupt IN)


Next, you must register the event handler on the [**UsbInterruptInPipe**](https://msdn.microsoft.com/library/windows/apps/dn278416) object that raises the [**DataReceived**](https://msdn.microsoft.com/library/windows/apps/dn278418) event.

This example code shows how to register the event handler. In this example, the class keeps track of the event handler, pipe for which the event handler is registered, and whether the pipe is currently receiving data. All that information is used for unregistering the event handler, shown in the next step.

```CSharp
private void RegisterForInterruptEvent(TypedEventHandler<UsbInterruptInPipe, UsbInterruptInEventArgs> eventHandler)
{
    // Search for the correct pipe that has the specified endpoint number
    interruptPipe = usbDevice.DefaultInterface.InterruptInPipes[0];

    // Save the interrupt handler so we can use it to unregister
    interruptEventHandler = eventHandler;

    interruptPipe.DataReceived += interruptEventHandler;

    registeredInterruptHandler = true;
}
```

```CSharp
void RegisterForInterruptEvent(TypedEventHandler<UsbInterruptInPipe, UsbInterruptInEventArgs> eventHandler)
    // Search for the correct pipe that has the specified endpoint number
    interruptInPipe = usbDevice.DefaultInterface.InterruptInPipes.GetAt(pipeIndex);
        
    // Save the token so we can unregister from the event later
    interruptEventHandler = interruptInPipe.DataReceived += eventHandler;

    registeredInterrupt = true;    

}
```

After the event handler is registered, it is invoked each time data is received in the associated interrupt pipe.

## <a href="" id="step-4--unregister-the--event-handler-to-stop-receiving-data--interrupt-in-"></a>Step 4: Unregister the event handler to stop receiving data (Interrupt IN)


After you are finished receiving data, unregister the event handler.

This example code shows how to unregister the event handler. In this example, if the app has an previously registered event handler, the method gets the tracked event handler, and unregisters it on the interrupt pipe.

```CSharp
private void UnregisterInterruptEventHandler()
{
    if (registeredInterruptHandler)
    {
        interruptPipe.DataReceived -= interruptEventHandler;

        registeredInterruptHandler = false;
    }
}
```

```CSharp
void UnregisterFromInterruptEvent(void)
{
    if (registeredInterrupt)
    {
        interruptInPipe.DataReceived -= eventHandler;

        registeredInterrupt = false;
    }
}
```

After the event handler is unregistered, the app stops receiving data from the interrupt pipe because the event handler is not invoked on interrupt events. This does not mean that the interrupt pipe stops getting data.

 

 


--------------------
[Send comments about this topic to Microsoft](mailto:wsddocfb@microsoft.com?subject=Documentation%20feedback%20%5Busbcon\buses%5D:%20How%20to%20send%20a%20USB%20interrupt%20transfer%20request%20%28Windows%20Store%20app%29%20%20RELEASE:%20%281/26/2017%29&body=%0A%0APRIVACY%20STATEMENT%0A%0AWe%20use%20your%20feedback%20to%20improve%20the%20documentation.%20We%20don't%20use%20your%20email%20address%20for%20any%20other%20purpose,%20and%20we'll%20remove%20your%20email%20address%20from%20our%20system%20after%20the%20issue%20that%20you're%20reporting%20is%20fixed.%20While%20we're%20working%20to%20fix%20this%20issue,%20we%20might%20send%20you%20an%20email%20message%20to%20ask%20for%20more%20info.%20Later,%20we%20might%20also%20send%20you%20an%20email%20message%20to%20let%20you%20know%20that%20we've%20addressed%20your%20feedback.%0A%0AFor%20more%20info%20about%20Microsoft's%20privacy%20policy,%20see%20http://privacy.microsoft.com/default.aspx. "Send comments about this topic to Microsoft")

