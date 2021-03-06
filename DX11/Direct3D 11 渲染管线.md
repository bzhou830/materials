## Direct3D 11 渲染管线

Direct3D 11渲染管线是使用GPU将内存资源处理为渲染图像的一种机制。管线本身由许多较小的逻辑单元组成，称为管线阶段（Pipeline Stage）。数据通过管线一次处理一个阶段，并在每个阶段以某种方式进行操作。通过了解管线的各个阶段是如何操作的，以及使用它们的语义，我们可以将管线作为一个整体来实现实时执行的各种算法。

随着GPU随着每一代新体系结构的出现而变得越来越强大，流水线的规模和能力都有了显著的扩展。此外，每个管线阶段的复杂性和可配置性稳步增加。当前渲染管线具有固定的功能阶段以及可编程着色器阶段。本章将首先考虑这些类型的管线阶段之间的差异。特别是，它将重点关注使它们运行所需的状态，以及它们可以执行的处理。在澄清这一区别之后，我们将考虑管线如何调用的更高级细节，以及每个流水线阶段如何与其相邻的阶段进行通信。

然后，我们将详细探讨每个管线阶段。这包括每个阶段执行的单个功能、阶段如何配置的详细讨论以及它们带来的一般语义。通过对管线的每一个单独组件的理解，我们可以考虑在各种配置中由流水线级组实现的一些高级功能。我们还将讨论整个管线的几个高级数据处理概念，包括如何管理这样一个复杂的处理体系结构。渲染管线已经演变成一套复杂的API，可用于实现多种算法。完成本章后，我们将深入透彻地了解如何使用渲染管线来开发用于实时渲染应用程序的高效有趣的渲染技术。


### 3.1 管线状态
从管线各阶段的名字就可以看出它是做什么的。数据在管道的一端作为输入提交，然后由第一个管道阶段进行处理。该数据由最多四个分量的基于向量的变量组成。第一阶段的处理完成后，修改后的输出数据将传递到下一阶段。然后将下一组数据带入第一阶段。这意味着前两个阶段同时处理不同的数据段。重复此过程，直到整个管道在输入数据的不同部分上同时运行。管道体系结构特别允许通过不同的管道阶段同时执行多个操作，这允许在单个数据项通过管道时对其执行许多专门的过程。一旦数据项到达管道的末端，它就会存储到输出资源中，稍后可以根据主机应用程序的需要使用该资源。这种流水线概念是一种简单但功能强大的处理技术，本质上类似于装配线。图3.1显示了管道如何处理数据。


开发人员的任务是正确配置管道的每个阶段，以便在数据从管道末端出现时获得所需的结果。管道配置过程通过操纵管道的每个单独阶段的状态来执行。通过将管道组织为多个阶段，Direct3D 11有效地将相关的状态集组合在一起，并整合它们的操作方式。有两种不同类型的管道级，固定功能级和可编程着色器级。这两种阶段类型都有一些关于数据如何流经它们以及应用程序如何操纵它们的状态的共同概念。以下各节将探讨这两种状态，并提供如何使用它们的一些一般概念。

#### 3.1.1 固定管道级
固定功能阶段对传递给它们的数据执行一组固定的操作。它们执行特定的操作，因此提供了可用功能的“固定”范围。它们可以以各种方式进行配置，但它们的性能始终相同管道
对传递给它们的数据的操作。这个概念的一个有用的类比是考虑正则函数如何在C++中工作。首先向函数传递预定义的参数列表。然后，它处理数据并将结果作为其输出返回。函数体表示固定函数阶段实现的操作，而输入参数是可用配置和实际输入数据。我们不能更改函数体，但可以控制在处理输入数据期间使用的选项。

前几代Direct3D中此类固定函数的一些示例包括更改顶点剔除顺序（现在是光栅化器阶段的一部分）、选择深度测试函数（现在是输出合并阶段的一部分）和设置alpha混合模式（也是输出合并阶段的一部分）的功能。这种对单个功能领域的关注主要是出于性能考虑。如果特定管道阶段是为特定任务设计的，则通常可以对其进行优化，以比通用解决方案更有效地执行该任务。

在Direct3D 9中，通过为每个要更改的单独设置调用API函数来更改这些固定函数阶段的状态。这需要执行许多API调用来为每个阶段配置特定的处理设置。根据场景内容和渲染配置，API调用的数量很容易增加，并开始导致性能问题。Direct3D 10引入了状态对象的概念，以替换这些单独的状态设置。状态对象用于通过单个API调用配置完整的功能状态。这大大减少了配置状态所需的API调用数量，还减少了运行时所需的错误检查量。Direct3D 11遵循这种状态对象范例。应用程序必须使用描述结构描述所需的状态，然后从中创建状态对象，该对象可用于控制固定功能管道阶段。

一次性创建完整状态的要求将状态验证从运行时API调用转移到状态创建方法。如果同时配置了不兼容状态，则会在创建时返回错误。创建后，状态对象是不可变的，不能修改。因此，无法为固定函数管道设置无效状态，这将有效地消除状态设置API调用的验证负担。总的来说，这种使用状态对象来表示管道状态的系统允许更精简的管道配置，只需要最少的应用程序交互。

#### 3.1.2 可编程管道级
可编程管道阶段包括管道的其余阶段。这里的“可编程”一词意味着这些阶段可以执行用高级着色语言（HLSL）编写的程序。通过使用上面的C++类比，可编程阶段允许您定义所需的输入参数和函数的主体。事实上，在这些阶段运行的程序是作为HLSL中的函数编写的。

执行程序的能力使这些管道阶段可用于各种各样的处理任务。这与固定功能阶段形成对比，固定功能阶段用于非常特定的任务，只提供少量的可配置性。在这些可编程管道阶段中执行的程序通常被称为着色器（shader）。当像素着色器阶段最初用于修改对象对照明的反应时，该名称继承自可编程性的早期步骤。随着越来越多的可编程阶段被添加到管道中，着色器的名称被用来指代所有可编程管道阶段。我们将在整本书中交替使用这个术语和可编程阶段。在以下各节中，我们将首先从较高的层次上了解这些可编程着色器阶段，然后在建立基本概念后深入了解其体系结构的细节。程序， 

**公共着色器核心**

所有可编程着色器阶段都建立在公共功能基础之上，称为公共着色器核心。公共着色器核心定义管道级的通用输入和输出设计，提供所有可编程着色器级支持的一组内在函数，以及可编程着色器可以使用的资源接口。图3.2提供了公共着色器核心操作方式的可视化表示。如上所述，数据流入舞台顶部，在着色器核心内进行处理，然后作为输出流出舞台

舞台的底部。在这些阶段中执行的着色器程序是为特定目的而用HLSL编写的函数。在着色器阶段内处理数据时，着色器程序可以访问由应用程序绑定到该阶段的常量缓冲区、采样器和着色器资源视图。一旦着色器程序完成了对其数据的处理，该数据将从后台传递出去，并引入下一段数据以重新开始该过程。

可编程管道级的可配置性不限于着色器程序内执行的处理。这些阶段的输入和输出数据结构也由着色器程序指定，该程序提供了一种从阶段到阶段传递数据的灵活方式。在这些阶段之间的接口上施加了一些规则，例如要求在特定阶段的接口中包含某些类型的数据。所需输出的一个典型示例是光栅化器之前的一个阶段提供位置输出，该位置输出可用于确定渲染目标的哪些像素被基本体覆盖。根据传递数据的阶段，参数要么未经修改提供，要么可以在传递到下一阶段之前进行插值。

虽然所有可编程阶段共享一组共同的功能，但每个阶段还可以提供其特有的其他专门特性和功能。这些通常与输入和输出语义相关，因此仅适用于特定阶段。当我们描述管道的每个阶段时，将在本章后面更详细地讨论每个阶段的单独行为。
由于具有如此大的灵活性和各种不同的管道阶段，有许多算法不需要所有阶段都处于活动状态。因此，可以通过清除其着色器程序来禁用可编程着色器阶段。此外，由于在每个阶段中进行的处理具有很大的灵活性，因此很可能在所有可编程阶段的不同组合中执行部分或全部所需工作。这种选择在何处执行特定计算的能力可用于我们的优势。如果可以在任何数据放大之前进行计算，则可以用少得多的操作计算相同的数据。我们将在第8章“网格渲染”中看到这方面的一个很好的示例，其中在对模型进行细分之前执行顶点蒙皮，以减少蒙皮顶点的数量。（有关顶点蒙皮的详细信息，请参见第8章。）

**着色器核心体系结构**

在上一节中，我们已经了解了可编程阶段如何运行的一般概念。它接收来自前一个阶段的输入，在其上执行HLSL程序，并将结果传递到下一个管道阶段。然而，这只是对实际情况的概述。在可编程阶段执行的着色器程序实际上是从HLSL编译成基于向量寄存器的汇编语言，设计用于GPU中的专用着色器处理器核心。即使所有着色器程序

 

图3.3 公共着色器核心的汇编语言视图。


必须用HLSL编写，但在渲染管道中使用之前，它们仍必须编译为此程序集字节码。12

我们可以从这种汇编语言中学到很多信息。它定义了一组特定的寄存器，编译器可以使用这些寄存器将HLSL程序映射到汇编语言。寄存器通常是四分量向量寄存器，可以使用单个分量来提供标量寄存器功能。着色器核心有用于接收其输入数据的寄存器、用于执行计算的临时寄存器、用于与资源交互的寄存器以及用于将数据传递出后台的寄存器。汇编语言指令使用这些寄存器执行各自的操作。通过了解汇编程序如何使用这些寄存器，我们可以深入了解着色器阶段的操作方式。

为了开始，我们将考虑图3.2中所示的常见着色器核心概述，但这次我们将从汇编语言的角度来查看它。图3.3显示了公共着色器核心的组件版本。


> 1  编译HLSL程序生成的汇编程序不会直接在GPU中执行。视频驱动程序将其进一步处理为特定于机器的指令，这些指令可能因GPU而异。即便如此，汇编语言还是提供了一个共同的参考点，我们可以借助它深入了解着色器处理器的操作。
2  编译过程的细节可以在第6章“高级着色语言”中找到



如图3.3所示，着色器核心的输入在v#寄存器中提供。由于它们向后台提供输入，因此它们自然是只读的。当执行着色器程序时，其输入数据在v#寄存器中可用。读取数据后，可以对其进行操作并与其他数据以及任何中间数据组合e计算可以存储在r#和x#[n]寄存器中。这些寄存器称为临时寄存器，由于它们持有中间值，它们都可以被着色器程序读写。纹理寄存器（t#）、常量缓冲寄存器（cb#[n]）、即时常量缓冲寄存器（icb[index]）和无序访问寄存器（u#也可用作数据源。这些寄存器用于提供对设备内存资源的访问，如第2章所述，除无序访问寄存器外，它们都是只读的。最后，将传递到下一管道级的计算值写入输出寄存器（o#）。当着色器程序终止时，存储在输出寄存器中的值将传递到下一阶段的输入寄存器，并在下一阶段重复该过程。其他一些特殊用途寄存器仅在某些阶段可用，因此我们将推迟到本章后面讨论。3

通常，开发人员不需要检查已编译着色器程序的程序集列表，除非存在性能问题。这使得了解程序集指令如何操作的细节不那么重要。即使如此，了解基于程序集的世界的基本知识仍然很有帮助。例如，当开发人员定义对于着色器程序的输入和输出数据结构，他们必须知道每个阶段可以使用多少输入和输出向量的限制。这取决于该特定阶段可用的输入和输出寄存器的数量。类似地，Constant Buffer、Texture和UAV resource的可用数量是这是非常重要的信息，在我们进行管道阶段的每一次讨论时都应该加以考虑。4


**GPU架构**

即使有一个严格定义的汇编语言规范，实际的GPU硬件也不需要直接实现该规范。有许多不同的体系结构实现，不同的供应商之间可能会有很大的差异。事实上，即使是来自同一供应商的连续几代GPU硬件也可能会因rom。这使得预测给定着色器程序在当前或未来GPU上的执行效率变得非常困难。根据执行该程序的GPU的体系结构，一种特定的内存访问模式可能比另一种更有效，但在不同的体系结构中，情况可能相反。


>3 #符号表示有多个可用寄存器，这些寄存器由整数索引标识。例如，v0和vl是前两个可用的输入寄存器。
4 有关如何编译着色器并查看其程序集列表的详细信息，请参见第6章。

由于各种实现之间存在巨大的差异，因此在此处提供详细信息是不切实际的。我们邀请读者更详细地探讨这一引人入胜的主题，并在（Fatahalian）中提供一个良好的起点但是，我们仍然可以提供GPU是如何组织的一般概念，以便本书后面的讨论将有一个上下文基础。GPU已经演变成一个大规模并行处理器，容纳数百个单独的ALU处理核心。这些处理器可以运行自定义程序，并且可以访问大量内存，这允许高带宽数据传输。每个GPU都使用某种形式的内存缓存系统来减少内存请求的有效延迟，尽管缓存系统是特定于供应商的，其低级设计通常不会公开。在过去，有必要了解特定目标的各个体系结构细节这一趋势可能会持续一段时间，因为GPU仍在以非常快的速度发展。



### 3.2 管道执行

固定功能阶段与可编程阶段结合在一起，提供了一个有趣且多样的管道，我们可以使用该管道渲染图像。执行管道的过程包括配置所有管道阶段状态，将输入和输出资源绑定到管道，然后调用其中一个绘制方法进行绘制执行中。所有这些任务都通过`ID3DllDeviceContext`接口的方法执行。为生成渲染帧而执行的管道执行次数取决于应用程序和当前场景，但我们可以假设每个帧至少执行一次管道执行。

使用这些draw方法之一调用管道后，数据将从输入内存资源读入管道的第一阶段。从输入资源读取的数据量取决于用于调用管道的draw调用的类型和参数。每一段数据都经过处理并传递到下一个阶段，直到它到达管道的末端，在那里它将被写入一个输出内存资源。处理完这些绘制调用中的所有数据后，可以将生成的输出资源（通常是2D渲染目标）显示到输出窗口或保存到文件中。

在本章中，我们将看到如何配置管道的每个阶段，以及理解每个绘制方法之间的差异。有七种不同的绘制方法可用于调用渲染管道。这里列出它们是为了便于参考，并演示有相当广泛的方法可用于调用渲染管道。

```cpp
Draw(...)
DrawAuto(...)
Drawlndexed(...)
DrawIndexedInstanced(...)
DrawInstanced(...)
DrawIndexedInstancedIndirect(...)
DrawInstancedIndirect(...) 
```

这些方法中的每一种都指示管道以不同的方式解释其输入数据。每个方法还提供不同的参数，以进一步配置要处理的输入数据量和输入数据。一旦启动管道执行，draw方法中指定的输入数据将在被引入管道时逐段处理。

在规划将在一个管道执行期间执行的工作时，最重要的考虑因素之一是每个阶段（固定和可编程）具有不同的输入和输出类型。此外，使用这些类型的语义具有相对较大的性能影响。例如，在一个简单的渲染序列中，我们可以将四个顶点作为两个三角形提交到管道。这将导致四个顶点着色器调用，这些调用将其输出传递到光栅化器阶段，该阶段将生成大量片段。然后将这些片段传递到像素着色器，最后传递到输出合并。如果三角形靠近查看器，可能会生成许多片段，从而导致相应的大量像素着色器调用。在这种情况下，顶点的数量是恒定的，而片段的数量取决于场景的条件。也可能出现相反的情况，其中细分阶段产生可变数量的要光栅化的顶点，但对象在屏幕上保持相同的大小，因此像素数保持不变。在设计算法时必须考虑这一点，因为平衡GPU上的工作量是至关重要的。

在硬件上实际执行管道期间，每个单独的管道阶段必须在可用的GPU硬件上尽快执行其计算。在许多情况下，GPU可以在不同的处理器组中同时执行多种类型的计算。在上面的示例中，GPU可以处理第一个三角形并将其光栅化以生成片段。然后，它可以使用它的一些处理核心来处理像素着色器调用，并使用一些来执行其余的顶点着色器工作。另一方面，它可以先处理所有顶点，然后再切换到处理所有片段。不保证GPU处理数据的顺序，除非它遵守管道中呈现的逻辑顺序。

#### 3.2.1 阶段间沟通

由于数据由各个管道级处理，因此必须从一个级传输到下一个级。为此，每个可编程阶段定义其着色器程序所需的输入和输出。我们将这些输入和输出称为属性，属性由向量变量（1到4个元素）组成。每个向量中的所有元素都由一种标量类型组成（整型和浮点型都可用）。我们还可以将这些属性称为语义或绑定语义，即为每个输入/输出属性指定的基于文本的标识符的名称。一般来说，当我们通过这些名称中的任何一个引用这些类型的阶段到阶段变量时，应该从使用它们的上下文中清楚地看到。

着色器程序的输入属性定义该程序执行所需的信息。确切地说，输入属性是着色器程序函数的输入参数。同样，函数的返回值定义其输出属性。因此，每个前一阶段必须产生与下一阶段所需输入参数相匹配的输出参数。这些输入和输出属性是通过我们在“着色器核心体系结构”一节中讨论的输入和输出寄存器实现的。清单3.1中的简单示例提供了两个着色器函数的示例声明，每个函数定义其输入和输出。请注意，第一个函数的输出与第二个函数的输入相匹配。

```cpp
struct VS_INPUT
{
	float3 position : POSITION;
	float2 coords: TEXCOORDS;
	uint vertexID : SV_VertexID;
};

struct VS_OUTPUT
{
	float4 position : SV_POSITION;
	float4 color : COLOR;
};

VS_OUTPUT VSMAIN(in VS_INPUT v )
{
	// Perform vertex processing here...
}
float4 PSMAIN(in VS_OUTPUT input) : SV_Target
{
	// Perform pixel processing here...
}
```
清单3.1 两个着色器函数定义其必须匹配的输入和输出属性。

在每个函数的输入和输出声明中，使用结构定义将所有输入和输出分组在一起。输入和输出结构的每个属性都有一个关联的语义，一个文本名称，用于在阶段之间将两个数据项链接在一起。语义位于属性名称之后，后面是分号。您还可以在组件名称的左侧看到每个组件的类型声明，就像您在C/C++中看到的类似声明一样。
从本例中可以看出，除了`vertexID`参数之外，第一个着色器函数的输出几乎与第二个函数的输入匹配。此特定参数旁边列出的语义是`SV_VertexID`。除了清单3.1中所示的用户定义的语义属性外，还有一组参数可供可编程着色器在其输入/输出签名中使用。这些被称为系统值语义。它们表示运行时生成或使用的属性，具体取决于它们的声明位置以及使用的系统值语义。这些系统值语义提供了在`HLSL`程序中使用的有用信息，还用于向管道指示所需的值。系统值始终以`SV_u`作为前缀，表示它们具有特殊意义，并且不是用户定义的。我们将看到每个系统值，以及它们如何在管道的每个阶段中使用。

理解如何将数据提供给管道，并了解每个管道阶段的状态如何影响发生的处理类型，是成功使用渲染管道的关键。在以下章节中，我们将按照管道中出现的顺序详细检查每个管道阶段。


### 3.3 输入装配器

输入装配阶段是渲染管道中的第一站。这是一个固定的功能阶段，负责将所有顶点放在一起，这些顶点将在管道中进一步处理，因此得名。作为管道的入口点，输入装配程序必须创建具有下一管道阶段顶点着色器所需属性的顶点。从一个或多个顶点缓冲区资源组装顶点的过程实际上可以覆盖大量不同的配置，我们将在本节后面详细介绍。该应用程序为输入装配程序提供了使用顶点布局对象构建顶点的路线图，这也允许在创建顶点缓冲区资源时在应用程序端采用灵活的策略。图3.4突出显示了输入装配程序在管道中的位置。


除了构造输入顶点外，输入装配器还通过指定正在渲染的几何体的拓扑来确定这些顶点之间的连接方式。这将标识定义的基本体或控制点的类型

#### 3.3.1 输入汇编程序管道输入

在将数据传递到管道之前，输入汇编程序必须连接到适当的资源以获取输入几何数据。该数据由顶点数据和索引数据组成，它们都以缓冲区资源的形式提供给管道。以下两部分描述如何将这些缓冲区资源绑定到输入汇编程序。

**顶点缓冲区使用**

输入汇编器的主要职责是正确地组装顶点，以便在管道中进一步使用，因此需要从应用程序访问顶点数据。该数据存储在一个或多个顶点缓冲区资源中，可以以各种不同的方式组织布局。从概念上讲，我们可以将输入汇编程序视为具有多个输入插槽，这些插槽可以用顶点缓冲区资源填充。图3.5显示了这种可视化。

图3.5 输入汇编程序的可用输入插槽概述

每个插槽都可以使用包含完整顶点的一个或多个属性的顶点缓冲区资源填充。例如，一个顶点缓冲区可以包含顶点的位置数据，第二个顶点缓冲区可以包含所有法向量数据。也可以在结构数组中的单个顶点缓冲区中提供所有顶点数据。开发人员可以选择如何最好地组织输入数据。当前有16个可供应用程序配置的输入插槽，这允许为所需的顶点数据提供相当灵活的存储选项。使用单独存储的各个属性，可以动态决定特定渲染需要哪些顶点组件。通过消除顶点中未使用的属性，这可能会减少读取顶点数据所需的带宽。我们将在本章后面讨论如何在多个缓冲区中存储各种数据的细节。

为了将顶点缓冲区绑定到输入汇编阶段，我们使用设备上下文接口。正如我们在第1章中看到的，设备上下文是完整管道的主要接口。使用`ID3D11DeviceContext::IASetVertexBuffers`方法将多个顶点缓冲区绑定到输入汇编程序的过程如清单3.2所示。在调用set vertex buffer方法之前，列表中声明的每个变量都将填充相应的缓冲区引用、步长和偏移量。

```cpp
UINT StartSlot;
UINT NumBuffers;
ID3D11Buffer* aBuffers[D3D11_IA_VERTEX_INPUT_RES0URCE_SL0T_C0UNT];
UINT aStrides[D3Dll_IA_VERTEX_INPUT_RESOURCE_SLOT_COUNT];
UINT a0ffsets[D3Dll_IA_VERTEX_INPUT_RESOURCE_SLOT_COUNT];
// Fill in all data h e r e . . .
pContext->IASetVertexBuffers(StartSlot, NumBuffers, aBuffers, aStrides, a0ffsets);
```

清单3.2 如何将多个顶点缓冲区绑定到输入汇编程序阶段

StartSlot和NumBuffers参数中指定了要开始绑定到的第一个输入汇编程序顶点缓冲区插槽的索引以及应绑定的后续缓冲区插槽的数量。顶点缓冲区资源指针列表必须组合成一个连续数组，并在ppVertexBuffers参数中传递。最后两个参数pStrides和poffset允许应用程序指定每个顶点缓冲区的逐顶点步长，以及开始使用每个顶点缓冲区到所需位置的偏移量。这些数组元素中的每一个都将开始在StartSlot索引中使用，并将用于填充NumBuff ens数量的插槽。这意味着数组的大小和填充必须适当，否则函数将访问无效的内存位置。此外，要将顶点缓冲区绑定到输入汇编程序，不能绑定缓冲区以在管道中的另一个位置写入。这是一个合理的要求，因为它会导致看似不可预测的结果，同时读取和写入缓冲区。

**索引缓冲区使用情况**

 当单独获取顶点数据时，输入汇编程序将按照顶点数据在顶点缓冲区中出现的顺序对其进行解释。这意味着，如果顶点数据用于渲染三角形，则前三个顶点定义第一个三角形，然后是下一个三角形的下三个顶点，依此类推。在两个相邻三角形实际引用同一顶点的情况下，这可能会导致重复的顶点数据。事实上，大多数三角形网格通常都有大量的相邻三角形。在顶点缓冲区中多次指定单个顶点数据时，可能会浪费大量内存。这种低效率可以通过使用索引缓冲区来消除。索引缓冲区用于通过向顶点缓冲区提供索引偏移来定义每个图元的顶点顺序。在这种情况下，每个顶点可以指定一次，然后指向它的索引可以用于它所属的任何三角形。使用索引渲染时，三角形中要使用的三个顶点由索引缓冲区中的三个连续索引指定。5

 

要使用索引呈现，应用程序必须将索引缓冲区绑定到输入汇编程序。只有一个插槽可用于绑定索引缓冲区，并且它仅在一个索引绘制调用期间用于指定用于生成基本体的顶点顺序。要将索引缓冲区绑定到输入汇编程序，应用程序必须使用ID3DllDeviceContext:：IASetIndexBuffer方法。清单3.3演示了如何执行此绑定操作。该方法提供了三个简单的参数：指向索引缓冲区的指针、索引存储的格式（16位或32位无符号整数），最后是索引缓冲区中的偏移量，以便从中开始构建原语。

> 5 顶点和索引的顺序因不同的基本体类型而异。本节后面将更详细地描述这些。



清单3.3。如何将索引缓冲区绑定到输入汇编程序阶段。

 

顶点或索引缓冲区绑定到管道后，可以通过将空指针值设置到相应的输入槽来解除绑定。记住这一点很重要，尤其是在draw调用之间重新配置管道时。如果在以前的绘图调用中使用了多顶点缓冲区配置，而新的绘图调用使用的顶点缓冲区较少，则未使用的插槽应填充NULL以解除未使用缓冲区的绑定。这就是为什么顶点缓冲区绑定方法总是使用一个大小为顶点缓冲区最大数量的数组，并在填充所需配置之前将其初始化为空值。

#### 3.3.2 输入汇编程序状态配置

将适当的资源绑定到顶点和索引缓冲区插槽后，在使用输入汇编程序之前，必须在输入汇编程序中设置另外两个配置。第一个是Input Layout对象，输入汇编程序使用该对象了解从哪个输入槽读取逐顶点数据以构建完整的顶点。第二个参数是基本体拓扑，输入汇编程序应使用该拓扑来确定顶点如何组合成基本体。这两种状态与用于执行管道的绘制方法相结合，确定管道如何解释顶点和基本体数据。我们将详细探讨每一个状态，然后考虑它们如何与可用的绘图方法交互以产生不同的输入配置。


输入布局

输入布局对象可以看作是一个配方，告诉输入汇编程序如何创建顶点。每个顶点由向量属性集合组成，每个属性最多有四个组件。由于多达16个顶点缓冲区可用于绑定，输入汇编程序需要知道从何处读取这些组件，以及了解将它们放入最终组装顶点的顺序。输入布局对象将此信息提供给输入汇编程序。

要创建一个输入布局对象，应用程序必须创建一个`D3D11_INPUT_ELEMENT_DESC`结构数组，其中所需顶点的每个组件都有一个结构

配置D3D11_INPUT_ELEMENT_DESC的成员如清单3.4所示。我们将检查每个结构成员代表什么，以及它们如何定义输入汇编程序执行其工作所需的信息。

```cpp
typedef struct D3D11_INPUT_ELEMENT_DESC {
  LPCSTR                     SemanticName;
  UINT                       SemanticIndex;
  DXGI_FORMAT                Format;
  UINT                       InputSlot;
  UINT                       AlignedByteOffset;
  D3D11_INPUT_CLASSIFICATION InputSlotClass;
  UINT                       InstanceDataStepRate;
} D3D11_INPUT_ELEMENT_DESC;
```

<center>清单3.4 D3D11_INPUT_ELEMENT_DESC结构成员</center>



**逐顶点元素。**第一个参数SemanticName标识顶点属性的文本名称，该名称必须与顶点着色器程序中提供的相应名称匹配。顶点着色器的每个输入必须在其HLSL源代码中定义语义。该语义名称用于将顶点着色器输入与输入汇编程序提供的顶点数据相匹配。Semanticlndex是一个整数，它允许SemanticName被多次使用。例如，如果在单个顶点布局中使用多组纹理坐标，则每组坐标可以使用相同的SemanticName，但会使用增加的Semanticlndex值来区分它们。

 下一个结构成员是组件的格式。这将指定什么数据类型，以及属性中包含多少元素。可用的格式范围从1到4个元素，浮点和整数类型都可用。下一个参数是InputSlot，它指示应从16个顶点缓冲槽中读取此组件的数据。AlignedByteOffset表示此D3D11_INPUT_element_DESC描述的项到顶点缓冲区中第一个元素的偏移量。这告诉输入汇编程序顶点缓冲区中从何处开始读取输入数据。

 **每个实例元素。**最后两个结构成员声明了有关实例化的顶点功能。实例化绘制方法基本上使用单个绘制调用将模型提交到管道。然后多次“实例化”此模型，但在每个实例级别（而不是每个顶点级别）指定顶点数据的子集。当同一对象的多个副本必须在整个场景的不同位置渲染时，经常使用此选项。顶点格式将在其定义中包含一个变换矩阵，对于模型的每个实例，变换矩阵仅递增到下一个值。然后在顶点中提供模型所有实例的变换矩阵

缓冲区并绑定到输入汇编程序。执行绘制调用时，顶点数据一次提交给顶点着色器一个副本，并在模型实例之间更新变换矩阵。

 如果顶点组件提供每个实例的数据，则必须使用InputSlotClass参数指定该数据。在这种情况下，还可以选择仅在向管道提交一定数量的实例后，才递增到顶点缓冲区中的下一个元素。这在InstanceDataStepRate中指定，并为每个实例参数的更改频率提供粗略控制。由于这是在单个属性级别指定的，因此可以使用不同的属性以不同的速率前进到下一个数据成员。识别所有所需的顶点属性并填写描述结构数组后，应用程序可以使用ID3DllDevice:：CreateInputl_ayout（）方法创建ID3DllInputLayout对象。清单3.5演示了如何执行此过程。

 

清单3.5。创建输入布局对象。

  这里我们可以看到D3D11_INPUT_ELEMENT_DESC结构的数组被分配，然后从输入容器对象填充。ID3D11 Device:：CreateInputLayout（）的前两个参数提供描述数组和数组中存在的元素数。第三个和第四个参数分别是指向编译顶点着色器源代码所产生的着色器字节代码的指针以及该字节代码的大小。这用于将元素描述的输入数组与将用于向其提供数据的HLSL着色器进行比较。比较这两个对象

 

当着色器与特定输入布局一起使用时，创建输入布局可降低运行时所需的验证量。


本原拓扑


必须在输入汇编程序中设置的最终状态是基本拓扑。我们可以将输入汇编程序的工作视为两个任务：首先，它必须使用输入布局对象从提供的输入资源中组装顶点；其次，它必须将顶点流组织成一个基本体流。每个基本体使用一小组连续的顶点（其顺序由它们在顶点缓冲区中的出现顺序或索引渲染的索引缓冲区中的索引顺序决定）来定义组成它的组件。基本拓扑的常见示例有三角形列表、三角形带、线列表和线带。在三角形列表的示例中，将为顶点流中的每三个顶点创建一个基本体。

 

基本体拓扑设置告诉输入汇编程序如何从组装的顶点流构建各种基本体。每个基本体中使用的顶点数量，以及如何从组合的顶点流中选择顶点，是基本体拓扑设置的函数。清单3.6中提供了可用的基元类型，并演示了如何使用ID3DllDeviceContext:：IASetPrimitiveTopology（）方法设置基元拓扑。



清单3.6。可用的基本拓扑类型，以及如何使用设备上下文接口设置拓扑类型的示例。

 

如清单3.6所示，有多种可用的基本类型。如果您以前使用过Direct3D 10，则列表中的前九个条目看起来应该很熟悉，因为它们在Direct3D 11之前就已经使用过了。但是，列表的其余部分提供了一系列不同的控制点修补程序列表，每个列表中包含的控制点修补程序数量不同，最多32个。引入这些基本类型是为了支持Direct3D 11中引入的新细分阶段。为了更好地理解所有这些基本拓扑如何组织顶点流，我们将创建一个具有有序顶点序列的示例，然后检查每个拓扑类型如何使用该顶点流创建基本拓扑。图3.6显示了顶点的样本集，每个顶点根据其在顶点流中的位置进行编号。

 

**点原语。**点列表基本体拓扑是所有基本体类型中最简单的。顶点被分组为单顶点基本体，这意味着顶点流产生大小相等的基本体流。因此，原语的输出组看起来与图3.6中所示的相同。

 

**行原语。**“线列表”基本体拓扑稍微复杂一些，每对顶点都会生成一个“线”基本体。这本质上表明，对于要渲染的每条线，包含一条线的两个顶点必须包含在顶点缓冲区中。“线带基本体”拓扑提供了线基本体更紧凑的表示形式，



图3.6。用于创建各种基本体类型的多个顶点。

 

但它只能表示一个连接的行列表。输入汇编程序从顶点流中的前两个顶点生成第一行。前两个之后的每个顶点定义一条新线，其中该线的另一个顶点是流中的上一个顶点。这在要绘制的线端到端连接的情况下提供了更密集的顶点表示。这两种基本拓扑如图3.7所示。

 

**三角形原语。**标准三角形基本拓扑遵循与直线基本拓扑相同的范例。三角形列表拓扑从顶点流中的每三个顶点创建一个三角形基本体。“三角形带”从流中的前三个顶点创建三角形基本体，然后为每个后续顶点创建新的三角形基本体，使用流中的前两个顶点定义三角形的其余部分。与线带基本体一样，三角形带基本体拓扑只能表示三角形的连接“带”。这些基本拓扑及其顺序如图3.8所示。

 

图3.7，从输入顶点创建的线列表和线带原语。



图3.8。从输入顶点创建的三角形列表和三角形条原语。

 

具有邻接关系的基元。在某些情况下，希望在管道中使用相邻的基本信息。这意味着每个原语都将被提供给管道，并且可以访问紧邻它的原语。有四种不同的基本拓扑提供邻接信息：具有邻接的线列表、具有邻接的线带、具有邻接的三角形列表和具有邻接的三角形带。这些表示为管道提供了更多的信息，以便在原语级别进行处理，但每个原语的内存和带宽成本较高。图3.9和3.10显示了这些基本拓扑中的每一种。

 

控制点原语。为了便于在细分管道中使用高阶基本体，提供了其他基本体拓扑选择。每个数量的控制点有一个基本类型，范围从1到32。这提供了执行大量不同控制补丁方案的能力。图3.11显示了一个样本控制补丁原语。


图3.9。从输入顶点创建具有邻接关系的线列表和线带基本体。



图3.10。三角形列表基本体，具有从输入顶点创建的邻接。

 

图3.11。控制从输入顶点创建的面片基本体。



#### 3.3.3 输入汇编程序阶段处理

 

对于输入汇编程序的所有可用设置，有各种各样的配置，可以生成不同的输出数据，以便在管道的其余部分中使用。在研究这些配置产生的内容之前，我们将更仔细地了解输入汇编程序中正在做什么。

顶点流

我们已经了解了输入汇编程序如何通过从绑定顶点缓冲区读取数据并将其组合到各个顶点中来创建顶点流。此顶点流以两个有序序列之一生成顶点。顶点要么按其顶点缓冲区中的现有顺序保留，要么按指定顺序重新排列

通过索引缓冲区中的索引。使用哪个顺序的确定来自用于调用管道执行的draw调用的类型。无论排序来自何处，输入汇编程序都会生成一个顶点流来表示输入几何体。


原始河流

顶点流进一步细化为基本体流。管道的各个部分旨在仅对顶点（顶点着色器）、顶点和基本体（外壳着色器和域着色器）或完整基本体（几何体着色器）进行操作。在每种情况下，原语流都由选定的原语拓扑和用于启动管道执行的绘制调用类型决定。

绘制调用效果


那么，绘制调用产生的顶点和基本体流究竟如何受到所用调用类型的影响呢？这可以通过检查每种类型的绘图调用的输入汇编程序的输出来最好地证明。以下各节提供了有关每个绘制调用类的信息，以及它们对结果数据流的影响。


**标准绘图方法。**要考虑的最简单的绘制方法是DRAW（）和DrawAuto（）方法。这两种方法都会触发输入汇编器根据其输入布局设置来组装顶点。这将创建一个顶点流，然后将其传递到管道中。这些绘制调用生成的基本体信息由输入汇编程序中当前指定的基本体拓扑确定，基本体的构造取决于顶点缓冲区中顶点的顺序。这个过程可以看作是基本顶点和基元的构造过程。

 

**索引绘制方法。**基本绘制调用的第一个变体是添加索引渲染。正如我们已经讨论过的，索引渲染使用索引缓冲区的索引来确定哪些顶点用于构造基本体，而不是简单地使用顶点缓冲区中的顶点顺序。顶点流保持不变，但其中的顶点由索引缓冲区的索引选择。此外，基元流使用索引缓冲区顺序而不是顶点缓冲区顺序创建其基元。有几个draw调用支持索引呈现，包括DrawIndexed（）、DrawIndexedlnstanced（）和DrawIndexedlnstancedlndirect（）。

 

**实例绘制方法。**除了索引渲染，Direct3D 11还允许实例化渲染。在描述D3D11_INPUT_ELEMENT_DESC结构的可用成员时，我们简要讨论了实例渲染。如果顶点组件声明为逐实例组件，并且使用其中一个实例调用管道


在绘制方法中，顶点和基本体流的创建方式与基本渲染和索引渲染的创建方式基本相同。但是，顶点和基本体流对于对象的每个实例都是重复的，但每个完整实例的逐实例顶点组件都会更新。实例的数量由传递给draw调用的参数确定，每个实例的顶点组件从绑定到输入汇编程序的一个或多个顶点缓冲区中获取。


总的来说，使用实例渲染方法的结果是顶点和基本体流乘以正在渲染的实例数。有四个不同的实例绘制调用：DrawInstanced（）、DrawIndexedlnstanced（）、DrawInstancedlndirectQ和Drawlndexedlnstancedlndirect（）。

间接绘制方法。除了基本、索引和实例渲染方法外，还有另一种类型的绘制调用：间接渲染。这种技术实际上并不修改顶点和基本体流，但这里应该讨论它，以获得可用绘制方法的完整视图。相反，间接呈现方法允许将缓冲区资源作为绘制调用参数传递。缓冲区包含通常由应用程序传递的所需输入信息。间接渲染方法是DrawInstancedlndirectQ和DrawIndexedlnstancedlndirect（）。例如，Drawlnstancedlndirect（）方法的顶点数、起始顶点数、实例数和起始实例数将包含在缓冲区资源中，而不是直接传递给函数。


这些调用的目的是允许GPU填充缓冲区，然后缓冲区可以控制如何执行绘制序列。这将draw调用的控制转移到GPU而不是应用程序，并提供了使GPU在其操作中更加自治的第一步。但是，间接渲染操作不会修改顶点和基本体流的构造，它们只会修改将绘制方法参数传递到运行时的方式。


混合渲染绘制方法。正如我们所看到的，每一类渲染操作并不是相互排斥的。绘制方法的名称通常包括几种渲染技术，并提供多种不同的方式来执行管道。在每种混合情况下，生成的顶点和基本体流都是独立渲染类型的混合。


#### 3.3.4输入汇编程序管道输出


用户定义的属性


随着理解输入汇编器可以产生什么，以及如何操作它的输出，我们需要考虑输出流如何与管道的其余部分交互。用于输入汇编程序生成与所需顶点着色器，用于创建输入布局对象的D3D11_INPUT_ELEMENT_DESC结构数组必须与顶点着色器声明的输入签名匹配。因此，各个顶点属性必须与顶点着色器所期望的匹配，包括数据类型和语义名称。清单3.7显示了HLSL中的顶点着色器程序示例。

清单3.7。顶点着色器输入声明示例。


对于本例，应用程序将需要提供顶点数据，该数据由具有三个浮点元素的位置语义属性、具有两个浮点元素的TEXCOORDS语义组件和具有三个浮点元素的普通语义组件组成。创建输入布局对象有助于开发人员确保顶点数据和顶点着色器签名彼此匹配。由于顶点数据是由应用程序提供的，因此可以使用与特定顶点着色器一起使用的正确格式创建或加载顶点数据。然后顶点着色器使用这些输入来执行自己的计算，然后它还将定义一些输出属性，并将这些属性写出以供下一个活动管道阶段使用。这个过程对每个后续的管道阶段重复，每个阶段声明其输出签名，然后在其中提供其输出数据。输入汇编程序生成的输入数据的一些典型示例包括但不限于以下内容：



输入汇编程序还可以生成三种不同的系统值语义：SV_VertexID、SV_PrimitiveID和SV_InstanceID。这些系统值语义不是由用户提供的，而是由输入汇编程序在创建输出数据流时生成的。SV_VertexID系统值是一个无符号整数，它唯一标识集合顶点流中的每个顶点。它首先在顶点着色器中可用，并提供了一种稍后在管道中区分顶点的简单方法。SV_原语ID为原语流中的每个原语提供类似的唯一标识符。它首先在外壳着色器阶段可用，因为顶点着色器不使用基本体级别信息。最后，SV_InstanceID在实例化绘制调用中唯一标识几何体的每个实例。它首先可用于顶点着色器阶段。这三个系统值一起提供了一种相当广泛的方法来识别输入汇编程序生成的每个数据元素。

