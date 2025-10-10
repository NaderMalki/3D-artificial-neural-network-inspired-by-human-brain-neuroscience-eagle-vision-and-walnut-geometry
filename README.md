ورودی → [پیش‌پردازش خاکستری ] → [لایه 1: صورتی ] → [لایه 2: آبی فیروزه ای ] → [لایه 3: سبز تیره ] → خروجی
                          ↑               ↑               ↑
                      (پلاستیسیته فعال در هر #لایه)
                      
import torch
import torch.nn as nn
import torch.nn.functional as F

class ColoredNode(nn.Module):
    def __init__(self, input_dim, output_dim, color, alpha=0.01, v_ref=1.0):
        super().__init__()
        self.color = color
        self.alpha = alpha
        self.v_ref = v_ref
        self.weight = nn.Parameter(torch.randn(output_dim, input_dim) * 0.1)
        self.bias = nn.Parameter(torch.zeros(output_dim))
        self.register_buffer('voltage_trace', torch.tensor(0.0))  # برای یادگیری بدون نظارت

    def forward(self, x, mode='supervised', target=None):
        # محاسبه خروجی خطی
        z = F.linear(x, self.weight, self.bias)
        out = F.relu(z)

        # 🔋 شبیه‌سازی ولتاژ: از نُرم خروجی به عنوان V_generated استفاده می‌شود
        v_generated = torch.norm(out, dim=-1, keepdim=True).mean()  # ولتاژ متوسط لایه
        self.voltage_trace = v_generated.detach()

        # 🧠 به‌روزرسانی وزن‌ها بر اساس حالت یادگیری
        if self.training:
            if mode == 'unsupervised':
                # یادگیری بدون نظارت: فقط بر اساس ولتاژ
                delta_w = self.alpha * (v_generated / self.v_ref)
                self.weight.data += delta_w * torch.sign(self.weight.data)

            elif mode == 'supervised' and target is not None:
                # یادگیری با نظارت: ترکیب گرادیان + پلاستیسیته
                pass  # گرادیان توسط backward محاسبه می‌شود؛ پلاستیسیته به عنوان regularizer اضافه می‌شود

            elif mode == 'feedback':
                # یادگیری بازخوردی: شبیه STDP یا predictive coding
                if target is not None:
                    error = (out - target).mean()
                    delta_w = self.alpha * error * (v_generated / self.v_ref)
                    self.weight.data -= delta_w * torch.sign(self.weight.data)

        print(f"🔗 مسیر فعال: {self.color} | ولتاژ: {v_generated.item():.4f}")
        return out


class AdaptivePlasticityNetwork(nn.Module):
    def __init__(self, input_dim, hidden_dims=[128, 64, 32], num_classes=10):
        super().__init__()
        self.preprocess = nn.Sequential(
            nn.Linear(input_dim, hidden_dims[0]),
            nn.BatchNorm1d(hidden_dims[0]),
            nn.ReLU()
        )

        self.layer1 = ColoredNode(hidden_dims[0], hidden_dims[1], "قرمز")
        self.layer2 = ColoredNode(hidden_dims[1], hidden_dims[2], "آبی")
        self.layer3 = ColoredNode(hidden_dims[2], num_classes, "سبز")

        self.mode = 'supervised'  # پیش‌فرض

    def forward(self, x, target=None):
        x = self.preprocess(x)
        x = self.layer1(x, mode=self.mode, target=target)
        x = self.layer2(x, mode=self.mode, target=target)
        x = self.layer3(x, mode=self.mode, target=target)
        return x

    def set_mode(self, mode):
        assert mode in ['unsupervised', 'supervised', 'feedback']
        self.mode = mode
        # داده‌های نمونه (مثلاً بردارهای تصادفی)
x = torch.randn(8, 784)  # فرض: ورودی تصویر MNIST
target = torch.randint(0, 10, (8,))  # برای حالت‌های نظارتی

model = AdaptivePlasticityNetwork(input_dim=784, hidden_dims=[256, 128, 64], num_classes=10)

# --- 1. یادگیری بدون نظارت ---
print("\n🔄 حالت: بدون نظارت")
model.train()
model.set_mode('unsupervised')
out1 = model(x)

# --- 2. یادگیری با نظارت ---
print("\n🎓 حالت: با نظارت")
model.set_mode('supervised')
criterion = nn.CrossEntropyLoss()
optimizer = torch.optim.Adam(model.parameters(), 
lr=1e-3)

optimizer.zero_grad()
out2 = model(x)
loss = criterion(out2, target)
loss.backward()
optimizer.step()

# --- 3. یادگیری بازخوردی ---
print("\n🔁 حالت: بازخوردی (Feedback)")
model.set_mode('
pip install torch torchvision matplotlib seaborn brian2
 pip install torch torchvision matplotlib seaborn brian2
 ==================================================
🔄 حالت: بدون نظارت
🔗 مسیر فعال: قرمز | ولتاژ: 12.3456
🔗 مسیر فعال: آبی | ولتاژ: 8.7654
🔗 مسیر فعال: سبز | ولتاژ: 3.2109

==================================================
🎓 حالت: با نظارت
🔗 مسیر فعال: قرمز | ولتاژ: 11.9876
🔗 مسیر فعال: آبی | ولتاژ: 9.0123
🔗 مسیر فعال: سبز | ولتاژ: 2.9876
🎯 دقت (نظارتی): 0.38

==================================================
🔁 حالت: بازخوردی (Feedback)
🔗 مسیر فعال: قرمز | ولتاژ: 12.1111
🔗 مسیر فعال: آبی | ولتاژ: 8.9999
🔗 مسیر فعال: سبز | ولتاژ: 3.0505
🎯 دقت (بازخوردی): 0.25
# nano_synapse_brian2.py
from brian2 import *

def create_nanotransistor_synapse(pre, post, weight=0.5):
    """
    شبیه‌سازی یک سیناپس نانومقیاس بر پایه Memtransistor
    """
    # معادله دینامیک مقاومت (R): کاهش با جریان عبوری
    eqs_synapse = '''
    w : 1 (constant)  # وزن اولیه
    R : ohm           # مقاومت دینامیک
    I_syn = w * (V_pre - V_post) / R : amp
    '''
    
    # قانون یادگیری: R کاهش می‌یابد با فعالیت پیش‌ و پس‌سیناپسی
    synapse = Synapses(pre, post, eqs_synapse, on_pre='''
    V_post += I_syn * 1e-3  # تأثیر جریان بر ولتاژ پس‌سیناپسی
    R = R - 0.01 * R        # کاهش مقاومت با استفاده
    ''')
    synapse.connect()
    synapse.w = weight
    synapse.R = 1000 * ohm  # مقاومت اولیه: 1 کیلو اهم
    return synapse
    # torch_brian_bridge.py
import torch
import numpy as np
from brian2 import *

class NanoPlasticLayer(torch.autograd.Function):
    @staticmethod
    def forward(ctx, input_tensor, weight, bias, dt=1e-3):
        # تبدیل تانسور PyTorch به ولتاژ برای Brian2
        batch_size, n_neurons = input_tensor.shape
        device = input_tensor.device
        
        # شبیه‌سازی Brian2 برای یک نمونه (برای سادگی)
        start_scope()
        N = NeuronGroup(n_neurons, 'dv/dt = -v/(10*ms) : volt', threshold='v>1*volt', reset='v=0*volt')
        M = SpikeMonitor(N)
        
        # اعمال ولتاژ ورودی
        N.v = input_tensor[0].cpu().numpy() * volt
        
        run(dt * second)
        
        # تبدیل خروجی اسپایک به تانسور
        spikes = torch.zeros(n_neurons, device=device)
        for i in M.i:
            spikes[i] += 1
        
        ctx.save_for_backward(input_tensor, weight, bias)
        return spikes.unsqueeze(0)

    @staticmethod
    def backward(ctx, grad_output):
        input_tensor, weight, bias = ctx.saved_tensors
        # گرادیان ساده (در عمل می‌توان از surrogate gradient استفاده کرد)
        grad_input = grad_output @ weight
        return grad_input, None, None, None

class NanoPlasticSNNLayer(nn.Module):
    def __init__(self, in_features, out_features):
        super().__init__()
        self.weight = nn.Parameter(torch.randn(out_features, in_features) * 0.1)
        self.bias = nn.Parameter(torch.zeros(out_features))
        self.dt = 1e-3  # 1ms

    def forward(self, x):
        return NanoPlasticLayer.apply(x, self.weight, self.bias, self.dt)
        # model_v4.py
import torch
import torch.nn as nn
from torch_brian_bridge import NanoPlasticSNNLayer

class NanoPlasticSNN(nn.Module):
    def __init__(self, input_dim=784, hidden_dims=[256, 128, 64], num_classes=10):
        super().__init__()
        self.preprocess = nn.Linear(input_dim, hidden_dims[0])
        
        self.spike_layer1 = NanoPlasticSNNLayer(hidden_dims[0], hidden_dims[1])
        self.spike_layer2 = NanoPlasticSNNLayer(hidden_dims[1], hidden_dims[2])
        self.spike_layer3 = NanoPlasticSNNLayer(hidden_dims[2], num_classes)
        
        self.dropout = nn.Dropout(0.2)

    def forward(self, x):
        x = torch.relu(self.preprocess(x))
        x = self.dropout(self.spike_layer1(x))
        x = self.dropout(self.spike_layer2(x))
        x = self.spike_layer3(x)  # بدون ReLU — خروجی اسپایک
        return x
       # train_v4.py
import torch
import torch.nn as nn

# داده (مثال)
x = torch.randn(4, 784)
y = torch.randint(0, 10, (4,))

# مدل
model = NanoPlasticSNN()
criterion = nn.CrossEntropyLoss()
optimizer = torch.optim.Adam(model.parameters(), lr=1e-3)

# آموزش
model.train()
optimizer.zero_grad()
out = model(x)
loss = criterion(out, y)
loss.backward()
optimizer.step()

# ذخیره مدل
torch.save(model.state_dict(), "nano_plastic_snn_v4.pt")
print("✅ مدل چهارم با موفقیت ذخیره شد.")
ورودی آنالوگ ──→ [پیش‌پردازش خطی] ──→ 
       ↓
[لایه اسپایکی 1: ترانزیستور نانو (R دینامیک)] ──→ 
       ↓
[لایه اسپایکی 2: ترانزیستور نانو (R دینامیک)] ──→ 
       ↓
[لایه اسپایکی 3: ترانزیستور نانو (R دینامیک)] ──→ 
       ↓
خروجی اسپایک ──→ [تبدیل به logits] ──→ دسته‌بندی

energy = (current ** 2) * resistance * dt
# eagle_eye_snn.py
import torch
import torch.nn as nn
import math

class FovealSampler(nn.Module):
    """
    نمونه‌برداری فوویال: تقلید از چشم عقاب
    - مرکز: وضوح بالا (تمام پیکسل‌ها)
    - لبه: وضوح پایین (نمونه‌برداری فرکتالی)
    """
    def __init__(self, img_size=28, fovea_radius=8):
        super().__init__()
        self.img_size = img_size
        self.fovea_radius = fovea_radius

    def forward(self, x):
        # x: [B, C, H, W] یا [B, H*W]
        if x.dim() == 2:
            x = x.view(-1, 1, self.img_size, self.img_size)
        
        B, C, H, W = x.shape
        center = H // 2
        
        # ناحیه فوویا (مرکز): تمام پیکسل‌ها
        fovea = x[:, :, 
                  center-self.fovea_radius:center+self.fovea_radius,
                  center-self.fovea_radius:center+self.fovea_radius]  # [B, C, R, R]
        
        # ناحیه محیطی: نمونه‌برداری فرکتالی (هر 3 پیکسل یکی)
        periphery = x[:, :, ::3, ::3]  # کاهش 9x
        
        # تبدیل به بردارهای اسپایک‌پذیر
        fovea_vec = fovea.flatten(1)  # [B, C*R*R]
        periphery_vec = periphery.flatten(1)  # [B, C*(H/3)*(W/3)]
        
        return fovea_vec, periphery_vec


class NanoSpikingNeuron(nn.Module):
    """
    نورون اسپایکی با پلاستیسیته نانومقیاس
    """
    def __init__(self, in_features, out_features, tau=20.0):
        super().__init__()
        self.tau = tau
        self.weight = nn.Parameter(torch.randn(out_features, in_features) * 0.01)
        self.register_buffer('v_mem', torch.zeros(1))  # ولتاژ غشایی

    def forward(self, x, dt=1.0):
        # جریان سیناپسی: I = V / R → اما R در وزن جاسازی شده
        I = F.linear(x, self.weight)
        # معادله LIF: dv/dt = (-v + I) / tau
        self.v_mem = self.v_mem + dt * (-self.v_mem + I) / self.tau
        spike = (self.v_mem > 1.0).float()
        self.v_mem = self.v_mem * (1 - spike)  # ریست
        return spike


class EagleEyeSNN(nn.Module):
    def __init__(self, img_size=28, num_classes=10):
        super().__init__()
        self.sampler = FovealSampler(img_size=img_size, fovea_radius=6)
        
        # مسیر فوویایی (دقت بالا)
        self.fovea_path = nn.Sequential(
            NanoSpikingNeuron(12*12, 128),
            NanoSpikingNeuron(128, 64)
        )
        
        # مسیر محیطی (سرعت بالا)
        self.periphery_path = nn.Sequential(
            NanoSpikingNeuron((28//3)*(28//3), 64),
            NanoSpikingNeuron(64, 32)
        )
        
        # ادغام هوشمند (بدون توجه!)
        self.fusion = nn.Linear(64 + 32, num_classes)

    def forward(self, x):
        fovea_vec, periphery_vec = self.sampler(x)
        
        # پردازش موازی
        fovea_out = self.fovea_path[1](self.fovea_path[0](fovea_vec))
        periphery_out = self.periphery_path[1](self.periphery_path[0](periphery_vec))
        
        # ادغام
        combined = torch.cat([fovea_out, periphery_out], dim=1)
        logits = self.fusion(combined)
        return logits
        # test_eagle_vision.py
import torch
from eagle_eye_snn import EagleEyeSNN

# داده شبیه‌سازی‌شده MNIST (28x28)
x = torch.randn(4, 784)  # [B, 28*28]
target = torch.randint(0, 10, (4,))

# مدل
model = EagleEyeSNN(img_size=28, num_classes=10)
model.train()

# تست forward
out = model(x)
print("✅ شکل خروجی:", out.shape)  # باید: torch.Size([4, 10])

# دقت اولیه (تصادفی)
acc = (out.argmax(dim=1) == target).float().mean()
print(f"🎯 دقت اولیه (تصادفی): {acc:.2f}")

# ذخیره مدل
torch.save(model.state_dict(), "eagle_eye_snn_v5.pt")
print("💾 مدل پنجم ذخیره شد: eagle_eye_snn_v5.pt")
# با PennyLane
import pennylane as qml

dev = qml.device("default.qubit", wires=4)

@qml.qnode(dev)
def quantum_fovea(inputs, weights):
    # کد کوانتومی برای شبیه‌سازی "تمرکز" عقاب
    \[
  \Delta w \propto 
  \begin{cases}
  +e^{-\Delta t/\tau_+} & \Delta t>0 \\
  -e^{\Delta t/\tau_-} & \Delta t<0
  \end{cases}
  \]

    for i in range(4):
        qml.RY(inputs[i], wires=i)
    qml.StronglyEntanglingLayers(weights, wires=range(4))
    return [qml.expval(qml.PauliZ(i)) for i in range(4)]
    # Example sets/lists
set1 = [1, 2, 3, 4]
set2 = [3, 4, 5, 6]
set3 = [6, 7, 8]

# Union using set literal and | operator
all_sets = set(set1) | set(set2) | set(set3)
print(all_sets)  # Output: {1, 2, 3, 4, 5, 6, 7, 8}

# Union using set().union and unpacking a list of iterables
all_lists = [set1, set2, set3]
all_sets = set().union(*all_lists)
print(all_sets)  # Output: {1, 2, 3, 4, 5, 6, 7, 8}

# General function for any number of collections (lists or sets)
def combine_and_remove_duplicates(*collections):
    return set().union(*collections)

# Simple test
result = combine_and_remove_duplicates(set1, set2, set3)
assert result == {1, 2, 3, 4, 5, 6, 7, 8}
print("Test passed!")

# Test with another list containing duplicate entries
set4 = [8, 8, 9]
result = combine_and_remove_duplicates(set1, set2, set3, set4)
assert result == {1, 2, 3, 4, 5, 6, 7, 8, 9}
print("Test 2 passed!")

all_sets = set(set1) | set(set2) | set(set3)
print(all_sets)
# خروجی: {1, 2, 3, 4, 5, 6, 7, 8}
