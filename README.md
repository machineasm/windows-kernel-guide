# Understanding Kernel Drivers - seized

# What is the Windows kernel?
## Introduction 

The Windows operating system relies on a complex architecture that includes the user space and the kernel space. The kernel is a critical component responsible for managing hardware resources, providing essential services, and ensuring the overall stability of the system. As a developer, understanding the basics of the Windows kernel is crucial for building robust and efficient applications. This document aims to provide an overview of key concepts related to the Windows kernel. 

## What is the Kernel? 

In computing, a kernel is the core part of the operating system that facilitates communication between software and hardware components. The Windows kernel is a fundamental layer that handles tasks such as process scheduling, memory management, device drivers, and system calls. 

## Key Components of the Windows Kernel 

1. Executive Services 

The Executive is a set of essential services that manage processes, memory, and I/O operations. It includes components like the Process Manager, Memory Manager, and I/O Manager. These services work together to ensure the smooth functioning of the operating system. 

2. Process Management 

The Process Manager oversees the creation, scheduling, and termination of processes. It allocates resources, such as CPU time and memory, to running applications. Developers interact with the Process Manager through APIs to create and control processes. 

3. Memory Management 

Memory Management is responsible for allocating and deallocating memory resources. It ensures that each process has the necessary memory space and protects processes from interfering with each other's memory. Understanding memory management is crucial for writing efficient and stable applications. Below is an example on how memory management could look like from a kernel driver.

![image](https://github.com/machineasm/windows-kernel-guide/assets/157221855/2b353f5c-e98b-4a76-9efb-309a4c446b89)

4. I/O Management 

The I/O Manager handles input and output operations, including interactions with devices such as disk drives and network interfaces. Developers use the Windows Driver Model (WDM) to create device drivers that communicate with the I/O Manager. 

5. System Calls 

System calls act as an interface between user-mode applications and the kernel. They allow applications to request services from the kernel, such as file operations, process creation, and communication between processes. 

6. Kernel Mode vs. User Mode 

Windows operates in two modes: Kernel Mode and User Mode. Kernel Mode has unrestricted access to hardware resources, while User Mode is more restricted. User-mode applications interact with the kernel through system calls, ensuring a separation between user and kernel spaces for security and stability. 

![image](https://github.com/machineasm/windows-kernel-guide/assets/157221855/cfa8e347-b5d3-4c3a-8158-4dbbd3012b38)

## Developing for the Windows Kernel 

Developers can extend or enhance the functionality of the Windows kernel through kernel-mode drivers. These drivers interact directly with the kernel and provide additional features or support for specific hardware devices. 

It is essential to adhere to Microsoft's guidelines and best practices when developing kernel-mode drivers to ensure compatibility, stability, and security. 

# Understanding Kernel Drivers - Anticheat Detections

## EasyAntiCheat Kernel Mode Detections

1. Manual Mapping Method:
You may be detected or kicked for not adequately clearing traces of your mapper or vulnerable driver. In the case of EAC, it is essential to clear PiDBBCache, MmUnloadedDrivers and WDFilters. While some claim that BattlEye (BE) doesn't check these, I still recommend clearing them to be cautious.

2. Big Pools:
EAC does have functions in its driver to scan for large memory pools, although it is unclear if they are actively banning for them. It is advisable to spoof the entry in the big pool table. Note that big pools are not necessary for allocating memory for your driver, but alternative methods should be explored.

3. Threads:
When using sockets or shared memory for communication, it is crucial to hide your threads. This can be achieved by spoofing the entry (replacing it with a dummy thread) or unlinking it from the list, though the latter may cause some issues. Additionally, hide the thread's call stack and spoof the start address. Avoid leaving any strings; instead, loop through entries and compare hashes. To prevent additional signature scans, randomize bytes in unused sections or mark them as a DISCARD section and erase them. *There are likely more methods, but these are the ones that immediately come to mind.*

4. Hooking:
If you intend to hook a syscall in win32kbase/ntoskrnl (.data pointer or vmt swap), route it to another memory destination located in a legitimate driver (forward it to a non-essential function, etc.), and then to your driver. While this step might not be necessary for BattlEye (BE), it is highly recommended for EAC. Although direct detection is uncertain, a recent update in December may have tapped into this method.

5. Common flags
- MmCopyVirtualMemory
- KeAttachProcess, KeDetachProcess, KeStackAttachProcess, KeUnstackDetachProcess
- Not setting the PFN's to zero
- Not clearing big pools

6. NMI Callbacks
Non-Maskable Interrupt (NMI) callbacks may be utilized as low-level system programming techniques to detect and respond to unauthorized activities by creating interrupts that cannot be ignored or disabled through normal means, thereby allowing for real-time monitoring and intervention at a kernel level to enhance the security and integrity of the game.

...and many more but these are the main detections of EasyAntiCheat.

- ## BattlEye Kernel Mode Detections
Please refer to this detailed UnknownCheats post: https://www.unknowncheats.me/forum/anti-cheat-bypass/505404-battleye-kernel-module-detection-depth-analysis.html

# Understanding Kernel Drivers - IOCTL's

In Windows, an Input/Output Control (IOCTL) driver is a type of device driver that allows applications to communicate with and control a device by sending control codes through I/O request packets (IRPs). These control codes are often defined by the device driver and are used to perform specific operations on the device.

## Device Initialization:

The driver registers itself with the Windows I/O manager during device initialization.
The driver specifies a Dispatch function to handle various types of IRPs, including IRPs with IOCTL requests.

## IOCTL Codes:

The IOCTL codes are numeric values that define the specific operation to be performed on the device.
IOCTL codes are often defined in header files and shared between the driver and user-mode applications.

## User-Mode Application Interaction:

User-mode applications use APIs like DeviceIoControl to communicate with the device driver.
These applications pass IOCTL codes along with input/output buffers to the driver.

## Dispatch Routine:

The driver's Dispatch routine handles IRPs with IOCTL requests.
The routine extracts the IOCTL code from the IRP and processes the request accordingly.
The driver may use switch statements or similar constructs to determine the action based on the IOCTL code.

## Processing IOCTL Requests:

The driver performs the necessary operations based on the IOCTL code.
Input and output buffers may be exchanged between the user-mode application and the driver to pass data.

```c++
#include <ntddk.h>

// Define IOCTL code (this is just an example)
#define IOCTL_MY_CUSTOM_OPERATION CTL_CODE(FILE_DEVICE_UNKNOWN, 0x800, METHOD_BUFFERED, FILE_ANY_ACCESS)

// Dispatch routine for handling IOCTL requests
NTSTATUS MyDriverDispatchIoctl(IN PDEVICE_OBJECT DeviceObject, IN PIRP Irp)
{
    UNREFERENCED_PARAMETER(DeviceObject);

    NTSTATUS status = STATUS_SUCCESS;
    PIO_STACK_LOCATION irpStack = IoGetCurrentIrpStackLocation(Irp);

    switch (irpStack->Parameters.DeviceIoControl.IoControlCode)
    {
    case IOCTL_MY_CUSTOM_OPERATION:
        // Handle custom IOCTL operation
        // Access input/output buffers using Irp->AssociatedIrp.SystemBuffer
        // Perform necessary operations, for example reading and writing
        break;

    default:
        status = STATUS_INVALID_PARAMETER;
        break;
    }

    // Complete the IRP
    Irp->IoStatus.Status = status;
    IoCompleteRequest(Irp, IO_NO_INCREMENT);

    return status;
}

// DriverEntry function
extern "C" NTSTATUS DriverEntry(IN PDRIVER_OBJECT DriverObject, IN PUNICODE_STRING RegistryPath)
{
    UNREFERENCED_PARAMETER(RegistryPath);

    DriverObject->MajorFunction[IRP_MJ_DEVICE_CONTROL] = MyDriverDispatchIoctl;

    // ... (other initialization code)

    return STATUS_SUCCESS;
}
```

