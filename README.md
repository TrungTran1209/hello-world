## Tạo và kích hoạt môi trường ảo venv
Mở thư mục gốc (Thư mục có chứa thư mục src)
```bash
py -m venv .venv
```
Window: 
```bash
.venv\Scripts\activate
```
macOS / Linux: 
```bash
source .venv/bin/activate
```
## Cài đặt các thư viện cần thiết
```bash
pip install -r requirements.txt
```
## Hướng dẫn chạy chương trình
1.  **Chuẩn bị:**
    Tải và mở thư mục gốc của dự án
2.  **Chạy chương trình chính:**
    Sử dụng lệnh sau để thực thi chương trình:

    ```bash
    python src/main.py
    ```
3.  **Kết quả:**
    Chương trình sẽ thực hiện lần lượt:
    * Phân tích (Parse) file PNML và kiểm tra tính nhất quán.
    * Tìm không gian trạng thái bằng thuật toán BFS (Task 2).
    * Tìm không gian trạng thái bằng thuật toán BDD (Task 3).
    * So sánh kết quả (số lượng trạng thái) và thời gian chạy giữa hai phương pháp.
    * Tìm kiếm trạng thái deadlock bằng BDD và ILP (Task 4).
# 📂 Cấu trúc Dự án

Mã nguồn được chia thành các module biệt lập để dễ quản lý và bảo trì:

* `main.py`: Điểm khởi chạy của chương trình. Điều phối việc đọc file, gọi các thuật toán và so sánh kết quả.
* `petrinet_model.py`: Định nghĩa các lớp dữ liệu (`Place`, `Transition`, `Arc`, `PetriNet`) và chứa logic kiểm tra tính nhất quán (`verify_consistency`).
* `parse_pnml.py`: Chứa hàm xử lý đọc file XML (`.pnml`) và chuyển đổi thành đối tượng `PetriNet`.
* `find_reachable_byBFS.py`: Cài đặt thuật toán tìm kiếm theo chiều rộng (BFS) để tính toán không gian trạng thái một cách tường minh (Task 2).
* `find_reachable_byBDD.py`: Cài đặt thuật toán tính toán tượng trưng sử dụng Binary Decision Diagrams (BDD) với thư viện `DD` (Task 3).
* `file.pnml`: File dữ liệu đầu vào mẫu (Mạng Petri).
* `deadlock_detection_by_ILP_BDD.py`: Cài đặt thuật toán xác định deadlock sử dụng ILP của thư viện guropy (Task 4)
* `optimization.py`: Cài đặt thuật toán để Optimized mạng Petri (Task 5)
* `generate_pnml.py`: File tạo sinh ra Mạng Petri ngẫu nhiên và lưu vào trong folder `/testcase` với chỉ mục tự động
* `test.py`: File tìm tất cả test được chứa trong `/testcase` và chạy 5 task với output được làm đẹp (task 4 vẫn chưa được format)
# ✨ Các tính năng đã thực hiện

* **Task 1: PNML Parsing**
    * Đọc file chuẩn PNML.
    * Xây dựng biểu diễn nội bộ (Internal Representation).
    * Kiểm tra tính nhất quán (Consistency Check) của mạng.
* **Task 2: Explicit Reachability (BFS)**
    * Duyệt đồ thị trạng thái bằng hàng đợi (Queue).
    * Lưu trữ trạng thái bằng `frozenset`.
* **Task 3: Symbolic Reachability (BDD)**
    * Mã hóa trạng thái và chuyển tiếp bằng biến Boolean.
    * Sử dụng thư viện `pyeda` để tính toán điểm bất động (Fixed-point iteration).
    * Tối ưu hóa tính toán trên không gian trạng thái lớn.
* **Task 4: Deadlock Detection (BDD and ILP Gurobi)**
    * BDD Phase: Tìm tập marking thỏa mãn điều kiện deadlock
    * Verification: Giao với tập reachable từ Task 3 (tránh spurious solutions)
    * ILP Phase: Sử dụng Gurobi để chọn deadlock tối ưu (nhiều token nhất)
* **Task 5: Optimization bằng phương pháp Cutting plane (BDD and ILP Gurobi)**
    * ILP formulation để tìm solution candidate
    * BDD verification để kiểm tra reachability
    * Cut generation để loại bỏ spurious solutions
    * Iterative refinement cho đến khi tìm được optimal valid solution

## 📘 Module: `petrinet_model.py`

Module này đóng vai trò là **xương sống (backbone)** của toàn bộ dự án. Nó định nghĩa các cấu trúc dữ liệu cốt lõi để biểu diễn một mạng Petri trong bộ nhớ máy tính.

Thay vì chỉ lưu trữ dữ liệu thô, module này được thiết kế tối ưu để hỗ trợ các thuật toán tìm kiếm (BFS, BDD) chạy hiệu quả nhất.

### 1. Các Lớp Cơ Bản (Basic Classes)
Đây là các thành phần nhỏ nhất cấu thành nên mạng:

* **`class Place`**: Đại diện cho một Vị trí.
    * Chứa `id` (định danh duy nhất).
    * Phương thức `__repr__` được ghi đè để hiển thị khi debug (ví dụ: `Place(id='p1')`).
* **`class Transition`**: Đại diện cho một Chuyển tiếp.
* **`class Arc`**: Đại diện cho một Cung nối, chứa thông tin `source_id` (nguồn) và `target_id` (đích).

### 2. Lớp Chính: `class PetriNet`
Lớp hoạt động như một "thùng chứa" (container) cho toàn bộ mạng lưới.

#### Cấu trúc dữ liệu (`__init__`)
Module duy trì hai hệ thống lưu trữ song song để tối ưu hóa hiệu năng truy xuất:

```python
# Tra cứu nhanh Place theo ID
self.places: Dict[str, Place] = {}   

# Tra cứu nhanh Transition theo ID
self.transitions: Dict[str, Transition] = {}

# Danh sách tuần tự (dùng cho in ấn/kiểm tra), không cần tra cứu bằng ID
self.arcs: list[Arc] = []               

# Tập hợp các Place có token ban đầu
self.initial_marking: Set[str] = set()   

# CẤU TRÚC TỐI ƯU CHO DUYỆT ĐỒ THỊ (O(1)):
self.pre: Dict[str, Set[str]] = {}       # Tập hợp đầu vào của mỗi Transition
self.post: Dict[str, Set[str]] = {}      # Tập hợp đầu ra của mỗi Transition
```
Lý do thiết kế:
* `self.pre` và `self.post` đóng vai trò như bảng tra cứu tốc độ cao. Thuật toán BFS/BDD có thể lấy ngay lập tức danh sách các Place cần thiết để kích hoạt một Transition mà không cần duyệt lại toàn bộ danh sách cung, giúp giảm độ phức tạp thuật toán.
* Sử dụng *Dictionary (Hash Map)* cho `places` và `transitions` để tối ưu hóa việc truy xuất theo ID về độ phức tạp $O(1)$, giúp quá trình parsing file lớn không bị chậm.
* Sử dụng *Set (Hash Set)* cho `pre` và `post` conditions để tận dụng các phép toán tập hợp mạnh mẽ của Python (`issubset`, `union`, `difference`), yếu tố cốt lõi giúp thuật toán BFS và BDD đạt hiệu suất cao.

#### Quản lý Chuyển tiếp (`add_transition`)
Hàm này thực hiện nhiều việc hơn là chỉ lưu trữ dữ liệu. Nó đóng vai trò **khởi tạo cấu trúc đồ thị**.

```python
def add_transition(self, trans: Transition):
    self.transitions[trans.id] = trans
    # KHỞI TẠO CẤU TRÚC ĐỒ THỊ:
    # Tạo trước các tập hợp rỗng cho đầu vào (pre) và đầu ra (post)
    self.pre[trans.id] = set()
    self.post[trans.id] = set()
```

#### Xử lý Logic Cung (`add_arc`)
Hàm chuyển đổi danh sách cung tuyến tính thành logic vận hành của mạng khi đọc file.
```python
def add_arc(self, arc: Arc):
    self.arcs.append(arc) # Lưu vào danh sách thô (để in ấn/debug)

    # PHÂN LOẠI VÀ XÂY DỰNG LOGIC:
    
    # 1. Cung đi từ Place -> Transition (Điều kiện kích hoạt - Pre)
    if arc.source_id in self.places and arc.target_id in self.transitions:
        self.pre[arc.target_id].add(arc.source_id)

    # 2. Cung đi từ Transition -> Place (Hệ quả kích hoạt - Post)
    elif arc.source_id in self.transitions and arc.target_id in self.places:
        self.post[arc.source_id].add(arc.target_id)
```
* **Cơ chế hoạt động:** Thay vì để các thuật toán (BFS/BDD) phải tự đi tìm xem "cung này nối cái gì với cái gì" mỗi lần chạy, hàm này phân loại ngay lập tức:
    * `self.pre[t]`: Chứa tất cả các token cần phải thu hồi khi `Transition` khai hỏa.
    * `self.post[t]`: Chứa tất cả các token sẽ được sinh ra khi `Transition` khai hỏa.

#### Thiết lập Trạng thái Đầu (`set_initial_marking`)
Hàm thiết lập điểm xuất phát ($M_0$) cho toàn bộ quá trình phân tích.
```python
def set_initial_marking(self, place_id: str):
    # Kiểm tra an toàn: Chỉ thêm nếu Place thực sự tồn tại
    if place_id in self.places:
        self.initial_marking.add(place_id)
    else:
        print(f"Cảnh báo: Đánh dấu ban đầu cho Vị trí không tồn tại '{place_id}'")
```
* **Tính chất**: Sử dụng `Set` để lưu trữ đảm bảo tính duy nhất. Trong mạng 1-safe, dù file input có lỡ khai báo trùng lặp (<initialMarking> xuất hiện 2 lần cho cùng 1 chỗ), logic chương trình vẫn đúng (chỉ lưu 1 lần).

#### Kiểm tra Nhất quán (`verify_consistency`)
Hàm này đóng vai trò là chốt chặn an toàn (Sanity Check) cho **Task 1**. Nó duyệt qua toàn bộ danh sách cung đã lưu trữ để kiểm tra tính toàn vẹn của cấu trúc mạng.

```python
def verify_consistency(self):
    # Tạo một tập hợp chứa tất cả các ID hợp lệ (Places + Transitions)
    all_nodes = self.places.keys() | self.transitions.keys()

    errors_found = False
    for arc in self.arcs:
        # Kiểm tra nguồn cung
        if arc.source_id not in all_nodes:
            print(f"Lỗi: Nguồn '{arc.source_id}' không tồn tại.")
            errors_found = True
            
        # Kiểm tra đích cung
        if arc.target_id not in all_nodes:
            print(f"Lỗi: Đích '{arc.target_id}' không tồn tại.")
            errors_found = True
            
    return not errors_found
```
Mục đích:
* Đảm bảo rằng mọi cung đều kết nối với các Nút (Node) thực sự tồn tại trong mạng.
* Ngăn chặn các lỗi "dangling pointer" (con trỏ treo) hoặc lỗi runtime khi chạy thuật toán duyệt đồ thị sau này.

## 📗 Module: `parse_pnml.py` 

Module này thực hiện **Task 1** của dự án, chuyển đổi dữ liệu thô từ file XML (`.pnml`) thành các đối tượng Python (`PetriNet`) mà chương trình có thể hiểu và xử lý được.

### Các thành phần chính

#### Thư viện sử dụng
* `xml.etree.ElementTree`: Thư viện chuẩn của Python dùng để phân tích cú pháp XML. Được chọn vì tính đơn giản, hiệu quả và không yêu cầu cài đặt thêm (built-in).

#### Hàm chính: `parse_pnml(file_path)`

Quy trình xử lý dữ liệu diễn ra theo các bước tuần tự sau:

1.  **Khởi tạo:**
    * Tạo một đối tượng `PetriNet` rỗng để chứa dữ liệu.
    * Sử dụng `ET.parse(file_path)` để đọc file và tạo cây XML (ElementTree).

2.  **Duyệt cây XML (Parsing Loop):**
    Hàm sử dụng phương thức `.findall()` với XPath (`.//tag`) để tìm kiếm các thẻ cần thiết bất kể chúng nằm sâu bao nhiêu trong cấu trúc file.

    * **Bước 2a - Đọc Vị trí (Places):**
        * Tìm tất cả thẻ `<place>`.
        * Lấy thuộc tính `id`.
        * Kiểm tra thẻ con `<initialMarking>` để xác định trạng thái đầu (token).
        * Gọi `net.add_place()` và `net.set_initial_marking()`.

    * **Bước 2b - Đọc Chuyển tiếp (Transitions):**
        * Tìm tất cả thẻ `<transition>`.
        * Lấy thuộc tính `id`.
        * Gọi `net.add_transition()`. *Lưu ý: Bước này cũng tự động khởi tạo cấu trúc đồ thị `pre`/`post` rỗng.*

    * **Bước 2c - Đọc Cung (Arcs):**
        * Tìm tất cả thẻ `<arc>`.
        * Lấy các thuộc tính `id`, `source`, `target`.
        * Gọi `net.add_arc()`. *Lưu ý: Bước này kích hoạt logic "biên dịch" để điền dữ liệu vào cấu trúc `pre`/`post`.*

3.  **Kiểm tra Nhất quán (Validation):**
    * Sau khi đọc xong, gọi `net.verify_consistency()` để đảm bảo dữ liệu toàn vẹn.
    * Nếu phát hiện lỗi (ví dụ: cung trỏ đến ID không tồn tại), hàm sẽ trả về `None` để ngăn chương trình chạy tiếp với dữ liệu sai.

4.  **Xử lý Lỗi (Error Handling):**
    * `ET.ParseError`: Bắt lỗi nếu file XML bị hỏng (cú pháp sai).
    * `FileNotFoundError`: Bắt lỗi nếu đường dẫn file không đúng.

## 📙 Module: `find_reachable_byBFS.py`

Module này thực hiện **Task 2** của dự án: Tính toán tường minh (Explicit) không gian trạng thái của mạng Petri bằng thuật toán **Breadth-First Search (BFS)**.

### Mục tiêu
Liệt kê toàn bộ các trạng thái (markings) có thể tới được từ trạng thái ban đầu bằng cách duyệt qua từng trạng thái một.

### Thiết kế Giải thuật

Thuật toán sử dụng mô hình "vết dầu loang" để khám phá không gian trạng thái.

#### 1. Khởi tạo
* Sử dụng `deque` (Double-ended Queue) từ thư viện `collections` để làm hàng đợi (queue) cho BFS. 
* Sử dụng `Set` để lưu trữ danh sách các trạng thái đã thăm (`visited`) tránh lặp vô hạn và giảm thời gian kiểm tra trùng lặp xuống $O(1)$.
* **Quan trọng:** Các trạng thái được lưu dưới dạng `frozenset`.
    * `set` trong Python là mutable (có thể thay đổi) nên không thể hash được và không thể thêm vào một `set` khác.
    * `frozenset` là immutable (bất biến), cho phép hash và lưu trữ an toàn trong tập `visited`.

#### 2. Vòng lặp BFS (BFS Loop)
Quá trình duyệt diễn ra như sau:

```python
while q:
    # 1. Lấy trạng thái đầu tiên ra khỏi hàng đợi
    current_marking = q.popleft()

    # 2. Duyệt qua TẤT CẢ các transition trong mạng
    for trans_id in net.transitions.keys():
        
        # 3. KIỂM TRA KÍCH HOẠT (Enabled Check)
        # Lấy tập hợp các Place đầu vào (Pre-set) của transition
        input_places = net.pre[trans_id]
        
        # Kiểm tra xem current_marking có chứa đủ các token đầu vào không
        # Sử dụng issubset() của Set (được tối ưu hóa cao trong CPython)
        if input_places.issubset(current_marking):
            
            # 4. KHAI HỎA (Fire) & TÍNH TRẠNG THÁI MỚI
            # Công thức: New = (Current - Pre) U Post
            output_places = net.post[trans_id]
            new_state = frozenset(current_marking - input_places | output_places)
            
            # 5. KIỂM TRA ĐÃ THĂM
            if new_state not in visited:
                q.append(new_state)
                visited.add(new_state)
```

## 📕 Module: `find_reachable_byBDD.py`

Module này thực hiện **Task 3: Tính toán Tượng trưng (Symbolic Reachability)**. Thay vì duyệt từng trạng thái riêng lẻ, thuật toán sử dụng **Binary Decision Diagrams (BDD)** để thao tác trên các tập hợp trạng thái khổng lồ cùng một lúc.


```python
import time
from functools import reduce
from dd.autoref import BDD # Sử dụng thư viện 'dd' (Pure Python)
```


### HÀM HỖ TRỢ: LOGIC CÂN BẰNG (BALANCED LOGIC)
`def balanced_op_iterative(manager, bdd_list, op_type="and")`

Thực hiện phép AND/OR trên danh sách BDD lớn.

- Tránh cây nghiêng (Skewed Tree): Vòng lặp tuần tự tạo ra cây biểu thức 
    lệch, không tối ưu cho BDD.
- Tránh lỗi đệ quy (RecursionError): Với hàng nghìn node (mạng lớn), 
    đệ quy thông thường sẽ gây tràn ngăn xếp Python.

- Sử dụng Hàng đợi (Queue) để gom cặp dần dần (Pairwise Merging).
- Đảm bảo độ sâu cây tính toán chỉ là O(log N).

### THUẬT TOÁN CHÍNH
```python
def find_reachable_byBDD(net: PetriNet):
    # A. KHỞI TẠO & CẤU HÌNH
    manager = BDD()
    
    # Tắt tính năng Sắp xếp động (Dynamic Reordering).
    # Lý do: Việc thư viện tự động dừng tính toán để sắp xếp lại biến tốn 
    # rất nhiều CPU. Ta sẽ chủ động sắp xếp tối ưu ngay từ đầu.
    manager.configure(reordering=False)

    # B. MÃ HÓA BIẾN (VARIABLE ENCODING)
    # Chiến lược: Interleaving Variable Ordering (Sắp xếp xen kẽ)
    # Thay vì khai báo hết v (v1..vn) rồi đến v' (v'1..v'n), ta khai báo:
    # v1, v'1, v2, v'2...
    # Lý do: Mối quan hệ giữ nguyên trạng thái (v == v') là phổ biến nhất.
    # Đặt chúng cạnh nhau giúp giảm kích thước BDD từ hàm mũ xuống tuyến tính.
    for place_id in sorted(net.places.keys()):
        p_prime = place_id + "_prime"
        manager.declare(place_id)
        manager.declare(p_prime)
        # ...

    # C. XÂY DỰNG QUAN HỆ CHUYỂN TIẾP (PARTITIONED TRANSITION RELATION)
    # Thay vì gộp tất cả thành một Monolithic TR khổng lồ (dễ tràn RAM),
    # ta xây dựng luật riêng cho từng Transition t:
    # TR_t = Pre_t(v) AND Post_t(v') AND Frame_t(v, v')
    
    # Tối ưu Frame Condition:
    # Tính trước bản đồ tương đương (Equivalence Map): (v <-> v')
    # Giúp tái sử dụng node, tránh tạo lặp lại hàng nghìn biểu thức giống nhau.
    equiv_map = {} 
    for p in place_ids:
        equiv_map[p] = (v[p] & v_prime[p]) | (~v[p] & ~v_prime[p])

    tr_list = []
    for trans_id in net.transitions:
        # ...
        # Pre-condition: Đầu vào phải có token
        pre_cond = manager.true
        for p in pre:
            pre_cond &= v[p]
        
        # Post-condition: Đầu ra nhận token, đầu vào mất token
        post_cond = manager.true
        for p in post:
            post_cond &= v_prime[p]  # p' = 1
        for p in pre_only:
            post_cond &= ~v_prime[p]  # p' = 0
        
        # Frame-condition: Các nơi KHÔNG liên quan phải giữ nguyên giá trị
        # Sử dụng Balanced AND để gộp hàng trăm điều kiện này lại
        frame_cond = bdd_and(manager, [equiv_map[p] for p in unchanged])
        
        tr_list.append(pre_cond & post_cond & frame_cond)

    # Gộp tất cả lại (Disjunction)
    TR = bdd_or(manager, tr_list)

    # D. VÒNG LẶP TÌM ĐIỂM BẤT ĐỘNG (FRONTIER SET FIXPOINT)
    # Chiến lược: Chỉ xử lý những trạng thái "Mới tìm thấy" (Frontier)
    
    Reachable = initial_state
    New_States = initial_state # Tập biên (Frontier)

    while True:
        # Bước 1: Tính ảnh (Image Computation)
        # Lọc ra các bước đi hợp lệ từ tập Frontier
        step_rel = New_States & TR
        if step_rel == manager.false: break 

        # Lượng từ tồn tại (Existential Quantification): 
        # Loại bỏ biến hiện tại (v), chỉ giữ lại biến tương lai (v')
        # (Đây là bước tốn kém nhất của thuật toán)
        current_step_image_prime = manager.exist(all_v_vars, step_rel)

        # Bước 2: Đổi tên biến (Renaming) v' -> v
        # Để chuẩn bị làm đầu vào cho vòng lặp kế tiếp
        New_v = manager.let(rename_map, current_step_image_prime)

        # Bước 3: Tính tập "Thực sự mới" (Difference)
        # Chỉ giữ lại những trạng thái chưa từng có trong Reachable
        # Phép toán này nhanh (O(1)) hơn so với so sánh Reachable == OldReachable
        really_new_states = New_v & ~Reachable
        
        if really_new_states == manager.false: 
            break # Dừng nếu không tìm thấy gì mới

        # Bước 4: Cập nhật
        Reachable = Reachable | really_new_states
        New_States = really_new_states # Cập nhật Frontier

    return Reachable
```
## 📕 Module: `deadlock_detection_by_ILP_BDD.py`
Module này thực hiện **Task 4: Phát hiện và lựa chọn deadlock marking tốt nhất**. <br>
Phương pháp: Kết hợp **Symbolic Reachability bằng Binary Decision Diagram (BDD) (Task 3)** và tối ưu hóa bằng **Integer Linear Programming (ILP – Gurobi)** <br>
<br>
Import các thư viện hỗ trợ và các module khác:
```python
import gurobipy as gp
from gurobipy import GRB
from dd.autoref import BDD
from petrinet_model import PetriNet
from find_reachable_byBDD import find_reachable_byBDD, bdd_and, bdd_or
```
### THUẬT TOÁN CHÍNH + Comment giải thích
```python
def check_deadlock(net, reachable_bdd, manager):
    """
    Task 4: Tìm Deadlock dùng BDD và ILP guropy.
    Sử dụng BDD cho reachabbility và ILP cho deadlock detection.
    Trả về dictionary {place_id: 1/0} giống format của ILP solver.
    """
    print("\n------ Start Deadlock Detection -------")

    # 1. Khai báo biến BDD (giống Task 3)
    place_ids = sorted(list(net.places.keys()))
    v = {p_id: manager.var(p_id) for p_id in place_ids}

    # 2. Xây dựng điều kiện Deadlock
    # Logic: t bị Disable nếu (p1=0) HOẶC (p2=0)... (Chỉ cần thiếu 1 cái là tạch)
    # ∀t ∈ T: ∑ M(p) ≤ |*t| - 1  (tương đương: có ít nhất 1 input place không có token)

    disabled_conditions = []

    for t_id in net.transitions.keys():
        pre_places = net.pre[t_id]

        # Nếu transition nguồn (không cần input) -> Luôn chạy -> Không bao giờ Dead
        if not pre_places:
            return None

        # FORMULATION: ∑ M(p) ≤ |*t| - 1 
        # Trong BDD: Có ít nhất 1 input place KHÔNG có token
        # => OR( NOT M(p) ) for p in *t
        disable_terms = [~v[p] for p in pre_places]  # place p ko có token -> p=0
        trans_disabled = bdd_or(manager, disable_terms)
        disabled_conditions.append(trans_disabled)

    if not disabled_conditions:
        # Không có transition nào -> Không có deadlock
        print("No transitions in the net with input place so cannot have deadlock.")
        return None

    # Trạng thái Dead = TẤT CẢ transition đều bị disable
    is_dead_bdd = bdd_and(manager, disabled_conditions)

    # VERIFICATION: Giao với tập reachable từ BDD Task 3
    # M_dead ∈ R_BDD (tránh deadlock giả)
    deadlock_candidates_bdd = reachable_bdd & is_dead_bdd

    if deadlock_candidates_bdd == manager.false:
        print("No deadlock found.")
        return None
    
    # 3. Lấy các deadlock candidates từ BDD để kiểm tra bằng ILP
    candidate_markings = []
    current_bdd = deadlock_candidates_bdd

    # Lấy tất cả các marking trong deadlock_candidates_bdd
    candidate_count = 0
    max_iterations = 1000  # Giới hạn số lượng để tránh quá tải

    while current_bdd != manager.false and candidate_count < max_iterations:
        # Lấy 1 marking từ BDD
        marking = manager.pick(current_bdd)
        candidate_markings.append(marking) 
        candidate_count += 1

        # Tạo điều kiện để loại bỏ marking này khỏi BDD
        # Tạo BDD cho marking cụ thể này để loại bỏ
        marking_bdd = manager.true
        for p_id in place_ids:
            if marking.get(p_id, False):
                marking_bdd &= v[p_id]
            else:
                marking_bdd &= ~v[p_id]
        
        # Loại bỏ marking này
        current_bdd &= ~marking_bdd
    
    if (candidate_count >= max_iterations):
        print(f"Warning: Reached maximum candidate extraction limit of {max_iterations}.")

    if (not candidate_markings):
        print("No deadlock found after extraction.")
        return None
    
    # Sắp xếp lại candidate markings để ưu tiên các marking có nhiều token hơn
    candidate_markings.sort(key=lambda m: sum(1 for p in place_ids if m.get(p, False)), reverse=True)
    
    # 4. Kiểm tra từng marking bằng ILP
    try:
        # Tạo model ILP bằng guropy
        model = gp.Model("deadlock_detection")
        model.setParam('OutputFlag', 0)  # Tắt output của guropy

        # Tạo biến quyết định cho TẤT CẢ candidate markings
        marking_vars = {}
        for i in range(len(candidate_markings)):
            marking_vars[i] = model.addVar(vtype=GRB.BINARY, name=f"select_marking_{i}")

        # Ràng buộc: Chỉ chọn 1 marking
        model.addConstr(gp.quicksum(marking_vars.values()) == 1, "Select_One_Marking")

        # Mục tiêu: tối đa hóa số token
        objective_terms = []

        # Nếu có weight cho places, dùng nó
        # Nếu ko có, dùng token làm mục tiêu

        use_weights = hasattr(list(net.places.values())[0], 'weight')
        
        if use_weights:
            # Tối ưu theo WEIGHT (nếu có)
            weights = {p_id: getattr(net.places[p_id], 'weight', 0) for p_id in place_ids}
            print("Using WEIGHTS for optimization objective")
        else:
            # Tối ưu theo SỐ TOKEN
            weights = {p_id: 1 for p_id in place_ids}  # Mỗi token = 1 điểm
            print("Using TOKEN COUNT for optimization objective")
        
        for i, marking in enumerate(candidate_markings):
            # Tính điểm cho marking này
            score = 0
            for p_id in place_ids:
                if marking.get(p_id, False):
                    score += weights[p_id]
            objective_terms.append(score * marking_vars[i])

        # Đặt hàm mục tiêu
        model.setObjective(gp.quicksum(objective_terms), GRB.MAXIMIZE)
        
        # Giải ILP
        model.optimize()

        # Kiểm tra kết quả
        if model.status == GRB.OPTIMAL:
            # Lấy marking được chọn
            for i, var in marking_vars.items():
                if var.X > 0.5:
                    selected_marking = candidate_markings[i]
                    
                    # Chuyển sang định dạng {place_id: 1/0}
                    deadlock_marking = {}
                    total_score = 0  # Tổng weight nếu có
                    for p_id in place_ids:
                        has_token = selected_marking.get(p_id, False)
                        deadlock_marking[p_id] = 1 if has_token else 0
                        if has_token:
                            total_score += weights[p_id]

                    # In kết quả
                    print("\nDEADLOCK MARKING:")

                    # In tất cả places theo nhóm
                    all_places_sorted = sorted(place_ids)
                    places_per_line = min(15, len(all_places_sorted))  # Số places mỗi dòng

                    for i in range(0, len(all_places_sorted), places_per_line):
                        chunk = all_places_sorted[i:i + places_per_line]
                        
                        # Dòng tên places
                        places_str = " ".join([f"{p:>4}" for p in chunk])
                        print(f"Place: {places_str}")
                        
                        # Dòng giá trị tokens
                        values_str = " ".join([f"{deadlock_marking.get(p, 0):>4}" for p in chunk])
                        print(f"Token: {values_str}")
                        print()

                    # Chỉ ra deadlock ở đâu
                    places_with_token = sorted([p for p in all_places_sorted if deadlock_marking.get(p, 0) == 1])
                    print("-" * 50)

                    if places_with_token:
                        # In thành nhiều dòng nếu nhiều places
                        places_str = ""
                        for i in range(0, len(places_with_token), 10):
                            chunk = places_with_token[i:i + 10]
                            places_str += ", ".join(chunk)
                            if i + 10 < len(places_with_token):
                                places_str += "\n" + " " * 22
                        
                        print(f"Deadlock at token=1 places: {places_str}")
                    else:
                        print("Deadlock: Empty marking (all places = 0)")

                    print("-" * 50)


                    return deadlock_marking

        else:
            print("No optimal solution found in ILP.")
            return None
    
    except gp.GurobiError as e:
        print("Gurobi Error:", e)
        return None
```
## 📕 Module: `optimization.py`
Module này thực hiện **Task 5: Optimization bằng phương pháp Cutting plane (BDD and ILP Gurobi).**
<br>
**Hiện thực**: 
<br>
 Tối ưu hóa marking dưới ràng buộc reachability, bằng cách kết hợp:
* **ILP(Gurobi)** để tìm tối ưu marking theo weight.
* **BDD** để xác minh marking có thực sự là reachable.
* **Cut Generation (Iterative Refinement)** để loại bỏ nghiệm sai.

```python
# Import các thư viện hỗ trợ 
import time
import gc
from typing import Dict, Tuple, Optional
```
**Các hàm hỗ trợ + Comment giải thích:**
```python
def build_incidence(net):
    """
    Xây dựng ma trận incidence C cho Petri net.
    - Với mỗi cặp (place p, transition t), ta xác định:
        pre(p,t)  = 1 nếu p là đầu vào của t, ngược lại 0
        post(p,t) = 1 nếu p là đầu ra của t, ngược lại 0
    - Giá trị C[(p, t)] = post(p, t) - pre(p, t)
        +1: firing t tạo token vào p
        -1: firing t tiêu thụ token từ p
         0: t không ảnh hưởng p
    Hàm trả về dictionary C dùng trong phương trình trạng thái của ILP:
        m = m0 + C * y
    """
    C = {}
    for p_id in net.places:
        for t_id in net.transitions:
            # post(p,t) = 1 nếu transition t tạo token cho p
            post = 1 if p_id in net.post.get(t_id, set()) else 0
            # pre(p,t) = 1 nếu transition t tiêu thụ token từ p
            pre = 1 if p_id in net.pre.get(t_id, set()) else 0
            C[(p_id, t_id)] = post - pre
    return C


def create_cube(manager, marking: Dict[str, int]):
    """
    Tạo một BDD 'cube' (biểu diễn đúng một trạng thái duy nhất) từ marking.
    - Marking là dict dạng: {place: 0/1}
    - BDD cube được dùng để kiểm tra reachability:
          reachable_bdd & cube != False  → marking reachable

    Vì một số backend BDD có thể yêu cầu dict phải 'sạch', hàm này thử:
        1. Tạo cube trực tiếp
        2. Nếu lỗi: tạo dict copy rồi thử lại
        3. Nếu vẫn lỗi: trả về None để xử lý bên ngoài
    """
    try:
        # thử tạo cube trực tiếp từ marking
        return manager.cube(marking)
    except Exception:
        try:
            # fallback: tạo bản sao dict để tránh lỗi reference
            return manager.cube({p: marking[p] for p in marking})
        except Exception:
            # không thể tạo cube
            return None

```
**THUẬT TOÁN CHÍNH + COMMENT:**
```python
def optimize_reachability(net, reachable_bdd, manager,
                          max_iter: int = 1000,
                          y_ub: Optional[int] = None
                          ) -> Tuple[Optional[Dict[str, int]], int, float, int]:
    """
    Tối ưu trên tập reachability bằng ILP + BDD verification + cắt loại nghiệm sai.
    Thuật toán:
        1. Giải ILP để tìm marking M* có tổng trọng số lớn nhất.
        2. Kiểm tra M* có reachable trong BDD hay không.
        3. Nếu không reachable → thêm ràng buộc cắt (spurious cut) để loại bỏ M*.
        4. Lặp lại cho đến khi:
            - Tìm được marking reachable tối ưu, hoặc
            - Hết lượt lặp.

    Trả về:
        (marking_tối_ưu, giá_trị_mục_tiêu, thời_gian, số_lượt_lặp)
    """

    start_time = time.time()

    # Danh sách place và transition được sắp theo thứ tự cố định
    place_ids = sorted(net.places)
    trans_ids = sorted(net.transitions)

    # Trọng số w(p) của mỗi place (do người dùng gán từ PNML)
    weights = {p: getattr(net.places[p], "weight", 0) for p in place_ids}

    # Marking ban đầu: 1 token nếu p nằm trong initial_marking
    m0 = {p: 1 if p in net.initial_marking else 0 for p in place_ids}

    # Ma trận incidence C(p,t)
    C = build_incidence(net)

    # Giới hạn trên cho số lần firing của transition
    if y_ub is None:
        y_ub = max(1, len(place_ids))

    # Khởi tạo mô hình ILP bằng Gurobi
    model = gp.Model("opt_over_reachable")
    model.Params.OutputFlag = 0    # tắt log
    model.Params.MIPGap = 0        # yêu cầu nghiệm tối ưu toàn cục

    # Biến marking nhị phân m_p ∈ {0,1}
    m_vars = {p: model.addVar(vtype=GRB.BINARY, name=f"m_{p}") for p in place_ids}

    # Biến firing count y_t ∈ [0, y_ub], nguyên
    y_vars = {t: model.addVar(vtype=GRB.INTEGER, lb=0, ub=y_ub, name=f"y_{t}") for t in trans_ids}

    # Mục tiêu: maximize Σ w(p) * m_p
    model.setObjective(gp.quicksum(weights[p] * m_vars[p] for p in place_ids), GRB.MAXIMIZE)

    # Phương trình trạng thái Petri net:
    #     m_p = m0_p + Σ C(p,t) * y_t
    for p in place_ids:
        model.addConstr(
            m_vars[p] == m0[p] + gp.quicksum(C[(p, t)] * y_vars[t] for t in trans_ids),
            name=f"state_eq_{p}"
        )

    iterations = 0
    added_cuts = 0

    # === Vòng lặp chính: ILP + BDD + Cut Generation ===
    while iterations < max_iter:
        iterations += 1
        model.optimize()

        # Nếu ILP không tìm được nghiệm → thoát
        if model.Status != GRB.OPTIMAL:
            duration = time.time() - start_time
            gc.collect()
            return None, 0, duration, iterations

        # M* là marking từ nghiệm ILP
        M_star = {p: 1 if m_vars[p].X >= 0.5 else 0 for p in place_ids}
        obj_val = sum(weights[p] * M_star[p] for p in place_ids)

        # Tạo cube BDD tương ứng để kiểm tra reachability
        cube_bdd = create_cube(manager, M_star)

        # reachable nếu: reachable_bdd ∧ cube != False
        is_reachable = (cube_bdd is not None) and ((reachable_bdd & cube_bdd) != manager.false)

        # Nếu marking là reachable → trả về nghiệm tối ưu
        if is_reachable:
            duration = time.time() - start_time
            gc.collect()
            return M_star, obj_val, duration, iterations

        # === Nếu marking KHÔNG reachable → tạo “spurious cut” ===
        added_cuts += 1

        # Tách place có token và không có token
        ones = [p for p, val in M_star.items() if val == 1]
        zeros = [p for p, val in M_star.items() if val == 0]

        # RHS đảm bảo nghiệm này bị loại nhưng không loại các nghiệm hợp lệ khác
        rhs = max(0, len(ones) - 1)

        # Ràng buộc cắt:
        #     Σ m_p (p∈ones) − Σ m_p (p∈zeros) ≤ RHS
        lhs = gp.quicksum(m_vars[p] for p in ones) - gp.quicksum(m_vars[p] for p in zeros)

        model.addConstr(lhs <= rhs, name=f"cut_spurious_{added_cuts}")

    # Nếu hết lượt lặp mà không tìm ra marking reachable
    duration = time.time() - start_time
    gc.collect()
    return None, 0, duration, iterations

```



