import random
import string
from datetime import datetime

BASE_GMAIL = "yourname@gmail.com"  # đổi thành Gmail thật của bạn

def make_alias(label="test"):
    name, domain = BASE_GMAIL.split("@")

    if domain.lower() != "gmail.com":
        raise ValueError("Email gốc phải là @gmail.com")

    suffix = "".join(random.choices(string.ascii_lowercase + string.digits, k=8))
    timestamp = datetime.now().strftime("%Y%m%d%H%M%S")

    return f"{name}+{label}-{timestamp}-{suffix}@gmail.com"

if __name__ == "__main__":
    print("Alias Gmail mới:")
    print(make_alias("signup"))# tooltaoemail
tool tạo email
