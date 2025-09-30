ورودی → [پیش‌پردازش] → [لایه 1: قرمز] → [لایه 2: آبی] → [لایه 3: سبز] → خروجی
                          ↑               ↑               ↑
                      (پلاستیسیته فعال در هر لایه)
                      
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
optimizer = torch.optim.Adam(model.parameters(), lr=1e-3)

optimizer.zero_grad()
out2 = model(x)
loss = criterion(out2, target)
loss.backward()
optimizer.step()

# --- 3. یادگیری بازخوردی ---
print("\n🔁 حالت: بازخوردی (Feedback)")
model.set_mode('
