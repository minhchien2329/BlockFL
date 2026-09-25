# BlockFL – Federated Learning tích hợp Blockchain

Đồ án học phần **Cơ sở Blockchain và Ứng dụng** – GVHD: Huỳnh Thế Thiện.

Đây là repo sạch để cả nhóm cùng code (mỗi đứa tự clone về, code phần mình rồi push), khác với bản nháp lúc đầu chỉ 1 người táy máy.

Ý tưởng chung của hệ thống, đi từ dưới lên:

```
IoT Edge Nodes (giả lập cảm biến vitals, chia data non-IID)
      │  mỗi node tự train local
      ▼
Federated Learning (FedAvg off-chain, gộp trọng số theo số mẫu)
      │  băm keccak256(Δw) + số mẫu
      ▼
Smart Contract (FederatedAggregator + BlockFLToken ERC-20)
      • submitWeights     → node nộp hash trọng số lên
      • aggregate         → owner chốt global model
      • distributeReward  → phát BFL token theo tỉ lệ đóng góp
```

Trọng số thật (`.npy`) thì lưu ở máy (`ai_model/weights/`), lên chain chỉ có mỗi cái hash 32 byte thôi — không đẩy data bệnh nhân lên đâu hết.

## Trong repo có gì

| Chỗ | Là gì |
|---|---|
| `contracts/BlockFLToken.sol` | Token ERC-20 (BFL) để thưởng, chỉ aggregator mint được |
| `contracts/FederatedAggregator.sol` | Contract chính: đăng ký node, nhận hash Δw, aggregate, chia thưởng |
| `test/aggregator.test.js` | 6 test Hardhat, test đủ luồng + phân quyền |
| `scripts/deploy.js` | Deploy contract + đăng ký node + ghi ra `deployments/<network>.json` |
| `scripts/web3_interface.py` | Python gọi thẳng vào contract qua Web3.py |
| `iot_code/data_partition.py` | Tạo data vitals giả + chia non-IID (Dirichlet) cho từng node |
| `iot_code/edge_node.py` | Class `EdgeNode` — train local + hash trọng số |
| `ai_model/model.py` | Model `VitalsMLP` (PyTorch) |
| `ai_model/local_train.py` | 1 bước train local (SGD) |
| `ai_model/fedavg.py` | Hàm FedAvg + hash keccak256 + lưu/đọc trọng số |
| `ai_model/baseline_centralized.py` | Baseline train tập trung (gom hết data 1 chỗ, không FL) để so sánh |
| `run_demo.py` | Chạy full demo nhiều round, có thể tắt chain để test nhanh |
| `dashboard/` | Web xem trực quan, đọc live từ contract |

Đúng cấu trúc repo thầy yêu cầu trong file hướng dẫn:

- `README.md` — cái file này, hướng dẫn cài đặt + chạy demo.
- `/contracts` — mã nguồn Smart Contract (Solidity).
- `/ai_model` — mã nguồn train AI + file trọng số.
- `/iot_code` — mã nguồn giả lập IoT.
- `Report_NhomXX.pdf` — báo cáo kỹ thuật chính thức (đang viết, chưa đưa vào repo).

## Cài đặt

Cần Node.js ≥ 18 với npm, và Python ≥ 3.10.

```bash
npm install
pip install -r requirements.txt
```

## Chạy demo

### 1. Test nhanh phần AI (chưa đụng blockchain)

```bash
python run_demo.py --no-chain --rounds 5
```

### 2. Test smart contract

```bash
npx hardhat test
```

### 3. Chạy full (AI + blockchain)

```bash
# mở 1 terminal, để chạy suốt lúc demo
npx hardhat node

# terminal khác
npx hardhat run scripts/deploy.js --network localhost
python run_demo.py --rounds 5
```

Mỗi round thì 4 node nộp hash trọng số lên, owner aggregate rồi phát thưởng BFL theo đúng tỉ lệ mẫu mỗi node đóng góp. Chạy xong in ra số dư token, ghi vào `results.json`.

### 4. Mở dashboard xem trực quan

Sau khi đã làm xong bước 3 (đang có hardhat node + đã deploy), mở thêm terminal:

```bash
python dashboard/serve.py 8000
```

Rồi vào `http://127.0.0.1:8000/dashboard/`. Có 8 trang, tự refresh mỗi 5s:

| Trang | Có gì |
|---|---|
| Tổng quan | round hiện tại, số node, tổng token, accuracy, cấu hình đang chạy |
| Kiến trúc CPS | sơ đồ 3 tầng, data đi đâu, 1 round chạy qua hàm nào |
| Dữ liệu IoT | data mỗi node lệch nhau sao (non-IID) |
| Hội tụ AI | accuracy/F1 tăng dần qua từng round, so với node tự train riêng |
| Round on-chain | lịch sử từng round đọc thẳng từ contract |
| Token thưởng | số dư BFL từng node |
| Nhật ký sự kiện | log mọi sự kiện on-chain |
| Smart Contract | trạng thái sống 2 contract, ai được gọi hàm nào |

Bên trang Tổng quan có nút **Chạy demo từ đầu** — bấm phát là nó tự reset chain, deploy lại, chạy FL thật luôn (chọn được số round 1-20 với seed), tầm 10-15 giây là xong. Muốn xem chi tiết thì mở song song trang Round on-chain hoặc Nhật ký sự kiện, nó hiện realtime.

Lưu ý: phải mở bằng `python dashboard/serve.py`, đừng mở thẳng file HTML là không chạy được đâu. Nút Kết nối MetaMask chỉ để xem số dư ví thôi, không bắt buộc phải có mới xem được dashboard.

### 5. Deploy lên Sepolia (không bắt buộc)

Thầy cho chọn Testnet hoặc Local, làm Local (Hardhat) như trên là đủ điều kiện rồi. Muốn làm thêm cho có điểm sáng tạo thì:

```bash
cp .env.example .env   # điền SEPOLIA_RPC_URL (xin ở Infura/Alchemy) và PRIVATE_KEY (ví MetaMask test)
npx hardhat run scripts/deploy.js --network sepolia
python run_demo.py --network sepolia --rounds 3
```

File `.env` đừng bao giờ push lên git (đã gitignore sẵn) — lỡ lộ private key là mất ví luôn đó.

## Mấy tham số của `run_demo.py`

| Cờ | Mặc định | Ý nghĩa |
|---|---|---|
| `--rounds` | 5 | chạy mấy round FL |
| `--nodes` | 4 | số node (chỉ áp dụng lúc `--no-chain`) |
| `--alpha` | 0.4 | Dirichlet α, càng nhỏ data càng lệch (non-IID mạnh) |
| `--epochs` | 10 | mỗi node train mấy epoch trước khi nộp Δw |
| `--lr` | 0.05 | learning rate |
| `--no-chain` | tắt | bật lên thì chỉ chạy FL, bỏ qua blockchain luôn |
| `--seed` | 7 | seed chia data — đổi seed là ra bộ data khác hẳn |

## So với train tập trung (không FL)

```bash
python ai_model/baseline_centralized.py --seed 4 --epochs 50
```

`--epochs 50` = 5 round × 10 epoch/round của FL, để so effort ngang nhau. Đổi epoch mặc định của `run_demo.py` thì nhớ đổi luôn số này (= 5 × epoch/round mới) cho khớp.
