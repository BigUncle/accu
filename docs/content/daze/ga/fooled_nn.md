# 愚弄神经网络

本篇文章灵感来自于上一篇[生物进化模拟](/content/daze/ga/evolve/), 那时思考, 如果使用一个训练好的图像分类神经网络作为遗传算法的适应度函数, 最终会生成什么图片呢? 比如有一个识别"猫"的神经网络, 我能否通过遗传算法凭空生成一张猫的图片? 我发现早在 2014 年就已经有研究者在研究这个问题了, 论文 [Deep Neural Networks are Easily Fooled:High Confidence Predictions for Unrecognizable Images](https://arxiv.org/pdf/1412.1897.pdf). 我决定使用 MNIST(手写数字数据集) 复现一下该研究. 研究目的就是通过遗传算法, 生成数字 0-9 的图片.

# 训练神经网络

这里使用 keras 来训练我们的模型. 直接用官方 examples 里的训练代码: [https://github.com/keras-team/keras/blob/master/examples/mnist_mlp.py](https://github.com/keras-team/keras/blob/master/examples/mnist_mlp.py), 记得在原始代码最后加上 `model.save_weights('mnist_mlp.h5')` 来保存模型到本地. keras 在该模型上给出的测试精度是 **98.40%**.

在完成训练后, 随机生成一个 28 * 28 的图片测试一下该模型:

```py
import keras.losses
import keras.models
import keras.optimizers
import numpy as np

model = keras.models.Sequential()
model.add(keras.layers.core.Dense(512, activation='relu', input_shape=(784, )))
model.add(keras.layers.core.Dropout(0.2))
model.add(keras.layers.core.Dense(512, activation='relu'))
model.add(keras.layers.core.Dropout(0.2))
model.add(keras.layers.core.Dense(10, activation='softmax'))
model.compile(
    loss=keras.losses.categorical_crossentropy,
    optimizer=keras.optimizers.RMSprop(),
    metrics=['accuracy']
)
model.load_weights('mnist_mlp.h5')


def predict(x):
    assert x.shape == (784, )
    y = model.predict(np.array([x]), verbose=0)
    return y[0]


x = np.random.randint(0, 2, size=784, dtype=np.bool)
r = predict(x)
print(r)
```

输出如下:

```
[  7.09424297e-09   0.00000000e+00   7.83010735e-04   0.00000000e+00
   0.00000000e+00   3.43550600e-14   9.99216914e-01   2.81605187e-19
   2.40218861e-36   2.99693766e-28]
```

# 开始调戏

代码和前几章基本一样, 唯一不同是使用神经网络作为遗传算法的适应度计算函数. 下示算法会初始化 80 张 28*28 的图片, 边将数据传入神经网络计算每张图片在某个数字上的得分, 如果在某一轮, 群体中最优秀的个体得分超过 0.99, 则结束进化, 并保存该最优群体.

```py
import os
import os.path

import keras.losses
import keras.models
import keras.optimizers
import numpy as np
import skimage.draw
import skimage.io
import skimage.transform

model = keras.models.Sequential()
model.add(keras.layers.core.Dense(512, activation='relu', input_shape=(784, )))
model.add(keras.layers.core.Dropout(0.2))
model.add(keras.layers.core.Dense(512, activation='relu'))
model.add(keras.layers.core.Dropout(0.2))
model.add(keras.layers.core.Dense(10, activation='softmax'))
model.compile(
    loss=keras.losses.categorical_crossentropy,
    optimizer=keras.optimizers.RMSprop(),
    metrics=['accuracy']
)
model.load_weights('mnist_mlp.h5')


class GA:
    def __init__(self, aim):
        self.aim = aim
        self.pop_size = 80
        self.dna_size = 28 * 28
        self.max_iter = 500
        self.pc = 0.6
        self.pm = 0.008

    def perfit(self, per):
        y = model.predict(np.array([per]), verbose=0)
        return y[0][self.aim]

    def getfit(self, pop):
        fit = np.zeros(self.pop_size)
        for i, per in enumerate(pop):
            fit[i] = self.perfit(per)
        return fit

    def genpop(self):
        return np.random.choice(np.array([0, 1]), (self.pop_size, self.dna_size)).astype(np.bool)

    def select(self, pop, fit):
        fit = fit - np.min(fit)
        fit = fit + np.max(fit) / 2 + 0.01
        idx = np.random.choice(np.arange(self.pop_size), size=self.pop_size, replace=True, p=fit / fit.sum())
        return pop[idx]

    def optret(self, f):
        def mt(*args, **kwargs):
            opt = None
            opf = None
            for pop, fit in f(*args, **kwargs):
                max_idx = np.argmax(fit)
                min_idx = np.argmax(fit)
                if opf is None or fit[max_idx] >= opf:
                    opt = pop[max_idx]
                    opf = fit[max_idx]
                else:
                    pop[min_idx] = opt
                    fit[min_idx] = opf
                yield pop, fit
        return mt

    def crosso(self, pop):
        for i in range(0, self.pop_size, 2):
            if np.random.random() < self.pc:
                a = pop[i]
                b = pop[i + 1]
                p = np.random.randint(1, self.dna_size)
                a[p:], b[p:] = b[p:], a[p:]
                pop[i] = a
                pop[i + 1] = b
        return pop

    def mutate(self, pop):
        mut = np.random.choice(np.array([0, 1]), pop.shape, p=[1 - self.pm, self.pm])
        pop = np.where(mut == 1, 1 - pop, pop)
        return pop

    def evolve(self):
        pop = self.genpop()
        pop_fit = self.getfit(pop)
        for _ in range(self.max_iter):
            chd = self.select(pop, pop_fit)
            chd = self.crosso(chd)
            chd = self.mutate(chd)
            chd_fit = self.getfit(chd)
            yield chd, chd_fit
            pop = chd
            pop_fit = chd_fit


save_dir = 'mnist_ga_fooled'

for n in range(10):
    ga = GA(n)
    for i, (pop, fit) in enumerate(ga.optret(ga.evolve)()):
        j = np.argmax(fit)
        per = pop[j]
        per_fit = fit[j]
        print(f'{n} {per_fit}')
        if per_fit > 0.99:
            skimage.io.imsave(os.path.join(save_dir, f'{n}.bmp'), per.reshape((28, 28)) * 255)
            break
```

在目录 `mnist_ga_fooled` 下保存了最终生成的数字 0-9 的图片, 每张图片在对应分类器下都有 99% 以上的概率. 结果有点有趣, 我们获得的是一推没有任何意义的噪点. 下示图片从左至右分别为数字 0-9.

![img](/img/daze/ga/fooled_nn/0.bmp)
![img](/img/daze/ga/fooled_nn/1.bmp)
![img](/img/daze/ga/fooled_nn/2.bmp)
![img](/img/daze/ga/fooled_nn/3.bmp)
![img](/img/daze/ga/fooled_nn/4.bmp)
![img](/img/daze/ga/fooled_nn/5.bmp)
![img](/img/daze/ga/fooled_nn/6.bmp)
![img](/img/daze/ga/fooled_nn/7.bmp)
![img](/img/daze/ga/fooled_nn/8.bmp)
![img](/img/daze/ga/fooled_nn/9.bmp)

愚弄神经网络真的很容易!
