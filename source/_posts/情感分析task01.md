---
title: 情感分析 | $\text{task01}$ 情感分析 $\text{baseline}$
date: 2021-09-15 23:50:34
tags: [NLP,情感分析]
categories: NLP 
---



开了一个新系列 准备系统学习下情感分析。 $\text{baseline}$ 使用了 $\text{torchtext}$ 进行数据处理，使用 $\text{RNN}$ 构建模型。

<!--more-->

```python
import torch
import random
import torch.nn as nn
import torch.optim as optim
import time
from torchtext.legacy import data
from torchtext.legacy import datasets

# 设置随机种子数，该数可以保证随机数是可重复的
SEED = 1234

# 设置种子
torch.manual_seed(SEED)
torch.backends.cudnn.deterministic = True

# 读取数据和标签
TEXT = data.Field(tokenize = 'spacy', tokenizer_language = 'en_core_web_sm')
LABEL = data.LabelField(dtype = torch.float)
train_data, test_data = datasets.IMDB.splits(TEXT, LABEL)

# 根据80%:20%构成训练集与验证集
train_data, valid_data = train_data.split(split_ratio=0.8 , random_state = random.seed(SEED))

print(f'Number of training examples: {len(train_data)}')
print(f'Number of testing examples: {len(test_data)}')

# vars()函数返回对象object的属性和属性值的字典对象
print(vars(test_data.examples[0]))

# 构造词汇表 (标记前max_size个)
# 只在训练集上 构造.
MAX_VOCAB_SIZE = 25000
TEXT.build_vocab(train_data, max_size = MAX_VOCAB_SIZE)
LABEL.build_vocab(train_data)

# 多了两个 <unk> <pad> 标签
print(f"Unique tokens in TEXT vocabulary: {len(TEXT.vocab)}")
print(f"Unique tokens in LABEL vocabulary: {len(LABEL.vocab)}")

# 创造迭代器 制作验证集 训练集 的batch数据
BATCH_SIZE = 64
device = torch.device('cuda' if torch.cuda.is_available() else 'cpu')
train_iterator, valid_iterator, test_iterator = data.BucketIterator.splits(
    (train_data, valid_data, test_data),
    batch_size = BATCH_SIZE,
    device = device)

#构建RNN
class RNN(nn.Module):
    def __init__(self, input_dim, embedding_dim, hidden_dim, output_dim):
        super().__init__()

        self.rnn = nn.RNN(embedding_dim, hidden_dim)
    #   nn.RNN(input_size, hidden_size, num_layers=1, nonlinearity=tanh, bias=True, batch_first=False, dropout=0, bidirectional=False)

        self.fc = nn.Linear(hidden_dim, output_dim)
    #   in_features指的是输入的二维张量的大小，即输入的[batch_size, size]中的size。
    #   out_features指的是输出的二维张量的大小，即输出的二维张量的形状为[batch_size，output_size]，当然，它也代表了该全连接层的神经元个数

        self.embedding = nn.Embedding(input_dim, embedding_dim)
    #   torch.nn.Embedding(num_embeddings, embedding_dim, padding_idx=None,
    #                      max_norm=None, norm_type=2.0, scale_grad_by_freq=False,
    #                      sparse=False, _weight=None)

    # 前向传播
    def forward(self, text):
        # text = [sent len, batch size]
        embedded = self.embedding(text)

        # embedded = [sent len, batch size, emb dim]
        output, hidden = self.rnn(embedded)

        # output = [sent len, batch size, hid dim]
        # hidden = [1, batch size, hid dim]
        assert torch.equal(output[-1, :, :], hidden.squeeze(0))
        return self.fc(hidden.squeeze(0))


def count_parameters(model):
    return sum(p.numel() for p in model.parameters() if p.requires_grad)

INPUT_DIM = len(TEXT.vocab)
EMBEDDING_DIM = 100
HIDDEN_DIM = 256
OUTPUT_DIM = 1
model = RNN(INPUT_DIM, EMBEDDING_DIM, HIDDEN_DIM, OUTPUT_DIM)

# print(f'The model has {count_parameters(model):,} trainable parameters')

# 优化器 (SGD 随机梯度下降)
optimizer = optim.SGD(model.parameters(), lr=1e-3)

# 损失函数 (BCEWithLogitsLoss 二元交叉熵 二分类)
criterion = nn.BCEWithLogitsLoss()
model = model.to(device)
criterion = criterion.to(device)

# 二分类准确率
def binary_accuracy(preds, y):
    """
    Returns accuracy per batch, i.e. if you get 8/10 right, this returns 0.8, NOT 8
    """

    #round predictions to the closest integer 四舍五入
    rounded_preds = torch.round(torch.sigmoid(preds))
    correct = (rounded_preds == y).float() #convert into float for division
    acc = correct.sum() / len(correct)
    return acc


def train(model, iterator, optimizer, criterion):
    epoch_loss = 0
    epoch_acc = 0
    #打开 dropout 和 batch normalization
    model.train()
    for batch in iterator:
        # 清除过往梯度
        optimizer.zero_grad()
        # 预测结果 (batch)
        predictions = model(batch.text).squeeze(1)
        # 计算损失值
        loss = criterion(predictions, batch.label)
        # 计算准确率
        acc = binary_accuracy(predictions, batch.label)
        # 反向传播
        loss.backward()
        # 更新梯度
        optimizer.step()
        epoch_loss += loss.item()
        epoch_acc += acc.item()

    return epoch_loss / len(iterator), epoch_acc / len(iterator)


def evaluate(model, iterator, criterion):
    epoch_loss = 0
    epoch_acc = 0
    # 关掉 dropout 和 batch normalization.
    model.eval()
    # 不用跟踪反向梯度
    with torch.no_grad():
        for batch in iterator:
            predictions = model(batch.text).squeeze(1)
            loss = criterion(predictions, batch.label)
            acc = binary_accuracy(predictions, batch.label)
            epoch_loss += loss.item()
            epoch_acc += acc.item()

    return epoch_loss / len(iterator), epoch_acc / len(iterator)

def epoch_time(start_time, end_time):
    elapsed_time = end_time - start_time
    elapsed_mins = int(elapsed_time / 60)
    elapsed_secs = int(elapsed_time - (elapsed_mins * 60))
    return elapsed_mins, elapsed_secs

print("Hello")

# 开始训练 训练多个epoch
N_EPOCHS = 1
best_valid_loss = float('inf')
for epoch in range(N_EPOCHS):
    start_time = time.time()
    train_loss, train_acc = train(model, train_iterator, optimizer, criterion)
    valid_loss, valid_acc = evaluate(model, valid_iterator, criterion)
    end_time = time.time()
    epoch_mins, epoch_secs = epoch_time(start_time, end_time)

    if valid_loss < best_valid_loss:
        best_valid_loss = valid_loss
        torch.save(model.state_dict(), 'tut1-model.pt')

    print(f'Epoch: {epoch + 1:02} | Epoch Time: {epoch_mins}m {epoch_secs}s')
    print(f'\tTrain Loss: {train_loss:.3f} | Train Acc: {train_acc * 100:.2f}%')
    print(f'\t Val. Loss: {valid_loss:.3f} |  Val. Acc: {valid_acc * 100:.2f}%')

model.load_state_dict(torch.load('tut1-model.pt'))
test_loss, test_acc = evaluate(model, test_iterator, criterion)
print(f'Test Loss: {test_loss:.3f} | Test Acc: {test_acc*100:.2f}%')
```

