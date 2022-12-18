---
title:  "[Intro to AI] CIFAR-10 AlexNet 학습"
date:   2020-12-18 09:00:00
header:
   overlay_image: /assets/static/header-yonsei.jpg
categories: 
   - Electronic Engineering
tags:
   - Electronic Engineering
   - Python
   - Net
toc: true
toc_sticky: true
breadcrumbs: true
---

# CIFAR-10 데이터셋 AlexNet 학습 시키기

Python 3.8, Jupyter 환경 이용하였습니다.

## Dataset 준비

**(a) Describe training setup e.g. data pre-processing/augmentation, initialization, hyperparameters such as learning rate schedule, momentum, dropout rate, batch number, BN etc. Present them systematically in the table form.**

The tables below shows the summarized specification of this design. First, the optimal methods used for this model found has the condition of:

| Pre-processing | Augmentation | LR Scheduler | Optimizer |
|:-----:|:-----:|:-----:|:-----:|
| Z-score <br>normalization | RandomCrop,<br> RandomHorizontalFlip,<br> ColorJitter, <br>RandomGrayscale | None | ADAM |

The hyperparameters are:

| Hyperparameter |      Setup     |
|:--------------:|:--------------:|
| Learning Rate  | 0.001          |
| Initialization | None, used default values |
| *betas* (ADAM) | Default (0.9, 0.999) |
| Dropout Rate   | 0.2           |
| Batch Number   | 512 |
| Classifier Neuron | 2048 |

The code written below describes the network design and its accuracy/loss. <u>The process made for environment selection will be discussed at *Question 1-(d)* with code attached.</u>  
  
The code fragment below shows the loading of dataset. To normalize the data using *transform* module, the following process must be gone through.  
1. Load data
2. Calculate means and standard deviation of each values: R, G and B. These values are in the range of 0 to 255.
3. Re-load the data, and prepare train-loader and test-loader based on the values prepared at step 2.

```python
import torch
import torchvision
import torchvision.transforms as transforms

import matplotlib.pyplot as plt
import numpy as np
import random

# First parameter: Batch size
BATCH_SIZE = 512

# Fills with nothing but the transform to tensor-function.
transform = transforms.Compose([
    transforms.ToTensor(),
])

# Calculate means and variance for normalization
trainset = torchvision.datasets.CIFAR10(root='./data', train=True,
                                        download=True, transform=transform)
trainloader = torch.utils.data.DataLoader(trainset, batch_size=BATCH_SIZE,
                                          shuffle=True, num_workers=4)

#
# Calculate each value, means and standard deviation
r_m, g_m, b_m = \
  np.mean(trainset.data[:, :, :, 0]) / 255, \
  np.mean(trainset.data[:, :, :, 1]) / 255, \
  np.mean(trainset.data[:, :, :, 2]) / 255

r_s, g_s, b_s = \
  np.std(trainset.data[:, :, :, 0]) / 255, \
  np.std(trainset.data[:, :, :, 1]) / 255, \
  np.std(trainset.data[:, :, :, 2]) / 255

print('Mean: %.3f, %.3f, %.3f' % (r_m, g_m, b_m))
print('Std: %.3f, %.3f, %.3f\n' % (r_s, g_s, b_s))

#
# Prepare for dataset.
transform = transforms.Compose([
    transforms.RandomCrop(32, padding=4),
    transforms.RandomHorizontalFlip(),
    transforms.ColorJitter(),
    transforms.RandomGrayscale(p=0.3),
    transforms.ToTensor(),
    transforms.Normalize((r_m, g_m, b_m), (r_s, g_s, b_s))
])

# There are several non-used transform methods. These will be discussed at (d).
# For the test dataset, shuffle option must be off, and other additional 
# transform must not be used.
transform_test = transforms.Compose([
    transforms.ToTensor(),
    transforms.Normalize((r_m, g_m, b_m), (r_s, g_s, b_s))
])

trainset = torchvision.datasets.CIFAR10(root='./data', train=True,
                                        download=True, transform=transform)
trainloader = torch.utils.data.DataLoader(trainset, batch_size=BATCH_SIZE,
                                          shuffle=True, num_workers=4)

testset = torchvision.datasets.CIFAR10(root='./data', train=False,
                                       download=True, transform=transform_test)
testloader = torch.utils.data.DataLoader(testset, batch_size=BATCH_SIZE,
                                         shuffle=False, num_workers=4)

# Print metadata and check the status
print("\n[Trainset Info]\n%s" % trainset)
print("\n[Trainloader Info]\n%s" % trainloader)
print("\n[Testset Info]\n%s" % testset)
print("\n[Testloader Info]\n%s\n" % testloader)

# Show randomly selected sample image
idx = random.randint(0,10)
tensor = trainset.__getitem__(idx)[0]
image = np.squeeze(tensor.numpy())
image = (image - np.min(image)) / (np.max(image) - np.min(image))
image = image.transpose((1, 2, 0))
plt.imshow(image)
```

Output:

```
Downloading https://www.cs.toronto.edu/~kriz/cifar-10-python.tar.gz to ./data/cifar-10-python.tar.gz

170500096/? [00:07<00:00, 23401452.89it/s]

Extracting ./data/cifar-10-python.tar.gz to ./data
Mean: 0.491, 0.482, 0.447
Std: 0.247, 0.243, 0.262

Files already downloaded and verified
Files already downloaded and verified

[Trainset Info]
Dataset CIFAR10
    Number of datapoints: 50000
    Root location: ./data
    Split: Train
    StandardTransform
Transform: Compose(
               RandomCrop(size=(32, 32), padding=4)
               RandomHorizontalFlip(p=0.5)
               ColorJitter(brightness=None, contrast=None, saturation=None, hue=None)
               RandomGrayscale(p=0.3)
               ToTensor()
               Normalize(mean=(0.49139967861519607, 0.48215840839460783, 0.44653091444546567), std=(0.2470322324632819, 0.24348512800005573, 0.26158784172796434))
           )

[Trainloader Info]
<torch.utils.data.dataloader.DataLoader object at 0x7fee9b36fb70>

[Testset Info]
Dataset CIFAR10
    Number of datapoints: 10000
    Root location: ./data
    Split: Test
    StandardTransform
Transform: Compose(
               ToTensor()
               Normalize(mean=(0.49139967861519607, 0.48215840839460783, 0.44653091444546567), std=(0.2470322324632819, 0.24348512800005573, 0.26158784172796434))
           )

[Testloader Info]
<torch.utils.data.dataloader.DataLoader object at 0x7fee9badc978>

<matplotlib.image.AxesImage at 0x7fee9953fac8>
```

The console output shows that there are 60,000 data in total. 50,000 for training set and 10,000 for testing set. To check whether the system successfully prepared the data, randomly selected image was shown using *matplotlib* library. It seems that the image was successfully loaded, and transformed by *RandomGrayscale* and *RandomCrop*.  

## Network 정의

Now, the structure below clearly shows the modified AlexNet. There are some variations made compared to the original AlexNet. The additional batch normalization layers were put just after each convolution layer along with two dropout layers at *classifier2*. 

```python
import torch.nn as nn

CLASSES = 10
DROPOUT_RATE = 0.2 # 0.2, 0.3

class AlexNet_SukJoon(nn.Module):
  def __init__(self, num_classes = CLASSES):
    super(AlexNet_SukJoon, self).__init__()

    self.features2 = nn.Sequential(
      nn.Conv2d(3, 64, kernel_size=3, stride=1, padding=2),
      nn.BatchNorm2d(64),
      nn.ReLU(inplace=True),
      nn.MaxPool2d(kernel_size=2),
      nn.Conv2d(64, 192, kernel_size=3, padding=2),
      nn.BatchNorm2d(192),
      nn.ReLU(inplace=True),
      nn.MaxPool2d(kernel_size=2),
      nn.Conv2d(192, 384, kernel_size=3, padding=1),
      nn.BatchNorm2d(384),
      nn.ReLU(inplace=True),
      nn.Conv2d(384, 256, kernel_size=3, padding=1),
      nn.BatchNorm2d(256),
      nn.ReLU(inplace=True),
      nn.Conv2d(256, 256, kernel_size=3, padding=1),
      nn.BatchNorm2d(256),
      nn.ReLU(inplace=True),
      nn.MaxPool2d(kernel_size=3, stride=2)
    )

    self.classifier2 = nn.Sequential(
      nn.Dropout(p=DROPOUT_RATE),
      nn.Linear(4096, 2048),
      nn.BatchNorm1d(2048),
      nn.ReLU(inplace=True),
      nn.Dropout(p=DROPOUT_RATE),
      nn.Linear(2048, 2048),
      nn.BatchNorm1d(2048),
      nn.ReLU(inplace=True),
      nn.Linear(2048, num_classes),
      nn.BatchNorm1d(10),
    )

  def forward(self, x):
    x = self.features2(x)
    x = x.view(x.size(0), -1)
    x = self.classifier2(x)
    return x
```

## 학습 시키기

Code segment below shows the training process. There are three inner for loop. Each represents <u>(1) training process</u>, <u>(2) measuring accuracy and loss for training set</u> and <u>(3) measuring accuracy and loss for testing set</u>.  

The code was ran using GPU, with the batch size of 512. The running time for 100 epoch was about an hour. The epoch size was not large enough, but the result shows reasonable accuracy. 

```python
from torch.autograd import Variable

device = torch.device("cuda:0" if torch.cuda.is_available() else "cpu")

net = AlexNet_SukJoon()
net = torch.nn.DataParallel(net)
net.to(device) # Present it to GPU

torch.cuda.empty_cache()

criterion = nn.CrossEntropyLoss()
optimizer = torch.optim.Adam(net.parameters(), lr=0.001)

print("[Device] {}\n".format(device))

# Variables for plots
acc_train = []
acc_test = []
loss_train = []
loss_test = []

end_epoch = 100
for epoch in range(1, end_epoch + 1):

  net.train()

  correct_test, total_test = 0, 0
  correct_train, total_train = 0, 0
  for i, data in enumerate(trainloader, 0):
    inputs, labels = data[0].to(device), data[1].to(device) # get the inputs
    inputs, labels = Variable(inputs), Variable(labels) # wrap them in Variable

    optimizer.zero_grad() # zero the parameter gradients
    outputs = net(inputs)

    # forward + backward + optimize
    loss = criterion(outputs, labels)
    loss.backward()
    optimizer.step()

  net.eval()
  with torch.no_grad():
    # Calculate loss and accuracy for training set
    for data in trainloader:
      inputs, labels = data
      inputs = inputs.cuda() # Give them to CUDA
      labels = labels.cuda()

      outputs = net(inputs)                          
      _, predicted = torch.max(outputs.data, 1)
      total_train += labels.size(0)
      correct_train += (predicted == labels).sum().item()

      loss = criterion(outputs, labels)
      loss_train.append(loss.item())

    # Calculate loss and accuracy for testing set
    for data in testloader:
      inputs, labels = data
      inputs = inputs.cuda()
      labels = labels.cuda()

      outputs = net(inputs)                          
      _, predicted = torch.max(outputs.data, 1)
      total_test += labels.size(0)
      correct_test += (predicted == labels).sum().item()

      loss = criterion(outputs, labels)
      loss_test.append(loss.item())

  # Gather Information
  acc_train.append(100 * correct_train / total_train)
  acc_test.append(100 * correct_test / total_test)

  if epoch % 5 == 0:
    print('\tEpoch: %3d/%d, Acc(train, test) : (%.2f%%, %.2f%%)' 
          % ( epoch, end_epoch, acc_train[-1], acc_test[-1]))

# Save the model
torch.save(net.state_dict(), 'alexnet_sukjoon.pth')
print('Done')
```

Output:
```
[Device] cuda:0

	Epoch:   5/100, Acc(train, test) : (78.28%, 78.76%)
	Epoch:  10/100, Acc(train, test) : (80.62%, 80.63%)
	Epoch:  15/100, Acc(train, test) : (88.65%, 86.13%)
	Epoch:  20/100, Acc(train, test) : (88.85%, 85.92%)
	Epoch:  25/100, Acc(train, test) : (93.24%, 88.16%)
	Epoch:  30/100, Acc(train, test) : (93.05%, 87.51%)
	Epoch:  35/100, Acc(train, test) : (95.17%, 89.18%)
	Epoch:  40/100, Acc(train, test) : (96.55%, 89.81%)
	Epoch:  45/100, Acc(train, test) : (96.92%, 89.44%)
	Epoch:  50/100, Acc(train, test) : (97.17%, 89.36%)
	Epoch:  55/100, Acc(train, test) : (97.56%, 89.59%)
	Epoch:  60/100, Acc(train, test) : (98.10%, 90.34%)
	Epoch:  65/100, Acc(train, test) : (98.19%, 90.27%)
	Epoch:  70/100, Acc(train, test) : (98.56%, 90.00%)
	Epoch:  75/100, Acc(train, test) : (98.47%, 90.00%)
	Epoch:  80/100, Acc(train, test) : (98.47%, 90.27%)
	Epoch:  85/100, Acc(train, test) : (98.63%, 89.87%)
	Epoch:  90/100, Acc(train, test) : (98.87%, 90.27%)
	Epoch:  95/100, Acc(train, test) : (98.45%, 90.01%)
	Epoch: 100/100, Acc(train, test) : (98.44%, 89.90%)
Done
```

The result shows that both accuracy gradually increased. They became over 75% by fifth epoch, and over 90% by 80th epoch. The console output shows that there are some points where accuracy falls slightly at some point compared to the former epoch. This is because the console prints out at every fifth epoch and the accuracy actually oscilates as epoch increases. It is shown at the graph, *Question 1-(c)*.  

The best training and test accuracy shown in this ouput is 98.87% and 90.34% each.

## Network 구조

**(b) Fill in the table below to show the weights and neurons of each layer in your modified AlexNet network.**

```python
# !pip install torchstat
from torchstat import stat
stat(net, (3, 32, 32))
```

Output:
```
         module name  input shape output shape      params memory(MB)           MAdd          Flops  MemRead(B)  MemWrite(B) duration[%]   MemR+W(B)
0        features2.0    3  32  32   64  34  34      1792.0       0.28    3,995,136.0    2,071,552.0     19456.0     295936.0       2.27%    315392.0
1        features2.1   64  34  34   64  34  34       128.0       0.28      295,936.0      147,968.0    296448.0     295936.0       1.82%    592384.0
2        features2.2   64  34  34   64  34  34         0.0       0.28       73,984.0       73,984.0    295936.0     295936.0       1.22%    591872.0
3        features2.3   64  34  34   64  17  17         0.0       0.07       55,488.0       73,984.0    295936.0      73984.0       2.08%    369920.0
4        features2.4   64  17  17  192  19  19    110784.0       0.26   79,847,424.0   39,993,024.0    517120.0     277248.0       5.01%    794368.0
5        features2.5  192  19  19  192  19  19       384.0       0.26      277,248.0      138,624.0    278784.0     277248.0       1.62%    556032.0
6        features2.6  192  19  19  192  19  19         0.0       0.26       69,312.0       69,312.0    277248.0     277248.0       1.09%    554496.0
7        features2.7  192  19  19  192   9   9         0.0       0.06       46,656.0       69,312.0    277248.0      62208.0       1.87%    339456.0
8        features2.8  192   9   9  384   9   9    663936.0       0.12  107,495,424.0   53,778,816.0   2717952.0     124416.0       6.47%   2842368.0
9        features2.9  384   9   9  384   9   9       768.0       0.12      124,416.0       62,208.0    127488.0     124416.0       1.64%    251904.0
10      features2.10  384   9   9  384   9   9         0.0       0.12       31,104.0       31,104.0    124416.0     124416.0       1.45%    248832.0
11      features2.11  384   9   9  256   9   9    884992.0       0.08  143,327,232.0   71,684,352.0   3664384.0      82944.0      14.07%   3747328.0
12      features2.12  256   9   9  256   9   9       512.0       0.08       82,944.0       41,472.0     84992.0      82944.0       2.40%    167936.0
13      features2.13  256   9   9  256   9   9         0.0       0.08       20,736.0       20,736.0     82944.0      82944.0       1.72%    165888.0
14      features2.14  256   9   9  256   9   9    590080.0       0.08   95,551,488.0   47,796,480.0   2443264.0      82944.0       6.53%   2526208.0
15      features2.15  256   9   9  256   9   9       512.0       0.08       82,944.0       41,472.0     84992.0      82944.0       1.40%    167936.0
16      features2.16  256   9   9  256   9   9         0.0       0.08       20,736.0       20,736.0     82944.0      82944.0       1.10%    165888.0
17      features2.17  256   9   9  256   4   4         0.0       0.02       32,768.0       20,736.0     82944.0      16384.0       1.55%     99328.0
18     classifier2.0         4096         4096         0.0       0.02            0.0            0.0         0.0          0.0       3.13%         0.0
19     classifier2.1         4096         2048   8390656.0       0.01   16,775,168.0    8,388,608.0  33579008.0       8192.0       8.10%  33587200.0
20     classifier2.2         2048         2048      4096.0       0.01            0.0            0.0         0.0          0.0       6.79%         0.0
21     classifier2.3         2048         2048         0.0       0.01        2,048.0        2,048.0      8192.0       8192.0       3.10%     16384.0
22     classifier2.4         2048         2048         0.0       0.01            0.0            0.0         0.0          0.0       5.65%         0.0
23     classifier2.5         2048         2048   4196352.0       0.01    8,386,560.0    4,194,304.0  16793600.0       8192.0       5.14%  16801792.0
24     classifier2.6         2048         2048      4096.0       0.01            0.0            0.0         0.0          0.0       3.24%         0.0
25     classifier2.7         2048         2048         0.0       0.01        2,048.0        2,048.0      8192.0       8192.0       2.20%     16384.0
26     classifier2.8         2048           10     20490.0       0.00       40,950.0       20,480.0     90152.0         40.0       2.76%     90192.0
27     classifier2.9           10           10        20.0       0.00            0.0            0.0         0.0          0.0       4.61%         0.0
total                                           14869598.0       2.69  456,637,750.0  228,743,360.0         0.0          0.0     100.00%  65009488.0
====================================================================================================================================================
Total params: 14,869,598
----------------------------------------------------------------------------------------------------------------------------------------------------
Total memory: 2.69MB
Total MAdd: 456.64MMAdd
Total Flops: 228.74MFlops
Total MemR+W: 62.0MB
```

| Layer | # of Filter / Filter Size | Stride / Zero Padding | Feature Map Size | # of Weights | # of Neurons | # of Conv Ops<br>(FLOPS) | Receptive Field | 
|:------:|:------:|:------:|:------:|:----------------:|:------:|:------:|:-----:|
| **Input** | - / - | - / - | $32\times32\times3$ | - | - | - | - |
| **Conv1** | $64$ / $3$ | $1$ / $2$ | $34\times34\times64$ | $1,792$ | $73,984$ | $2,071,552.0$ | $3$ |
| **BN1** | - / - | - / - | $34\times34\times64$ | $128$| - | $147,968.0$ | - |
| **Pool1** | - / $2$ | $2$ / - | $17\times17\times64$ | - | - | $73,984.0$ | - |
| **Conv2** | $192$ / $3$ | $1$ / $2$ | $19\times19\times192$ | $110,784$ | $69,312$ | $39,993,024.0$ | $5$ |
| **BN2** | - / - | - / - | $19\times19\times192$ | $384$ | - | $138,624.0$ | - |
| **Pool2** | - / $2$ | $2$ / - | $9\times9\times192$ | - | - | $69,312.0$ | - |
| **Conv3** | $384$ / $3$ | $1$ / $1$ | $9\times9\times384$ | $663,936$ | $31,104$ | $53,778,816.0$ | $7$ |
| **BN3** | - / - | - / - | $9\times9\times384$ | $768$ | - | $62,208.0$ | - |
| **Conv4** | $256$ / $3$ | $1$ / $1$ | $9\times9\times256$ | $884,992$ | $20,736$ | $71,684,352.0$ | $9$ |
| **BN4** | - / - | - / - | $9\times9\times256$ | $512$ | - | $41,472.0$ | - |
| **Conv5** | $256$ / $3$ | $1$ / $1$ | $9\times9\times256$ | $590,080$ | $20,736$ | $47,796,480.0$ | $11$ |
| **BN5** | - / - | - / - | $9\times9\times256$ | $512$ | - | $41,472.0$ | - |
| **Pool3** | - / $3$ | $3$ / - | $4\times4\times256$ | - | - | $20,736.0$ | - |
| **FC1** | - / - | - / - | $1\times2048$ | $8,390,656$ | $2,048$ | $8,388,608.0$ | - |
| **BN6** | - / - | - / - | $1\times2048$ | $4096$ | - | - | - |
| **FC2** | - / - | - / - | $1\times2048$ | $4,196,352$ | $2,048$ | $4,194,304.0$ | - |
| **BN7** | - / - | - / - | $1\times2048$ | $4096$ | - | - | - |
| **FC3** | - / - | - / - | $1\times10$ | $20,490$ | $10$ | $20,480.0$ | - |
| **BN8** | - / - | - / - | $1\times10$ | $20$ | - | - | - |
| **Total** |  - / - |  - / - |  - / - | $14,869,598$ | $219,978$ | $228,743,360.0$ | $11$ |


