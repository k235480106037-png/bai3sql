# BÀI TẬP 3: HỆ QUẢN TRỊ CƠ SỞ DỮ LIỆU
# Nhiệm vụ 1:
# Nhiệm vụ 2:
1. Tạo bảng cơ sở
```sql
Create Database QLCamDo
Go
-- 1. Bảng Khách hàng
CREATE TABLE KhachHang (
    KhachHangID INT IDENTITY(1,1) PRIMARY KEY,
    HoTen NVARCHAR(100) NOT NULL,
    SoDienThoai VARCHAR(15),
    CMND VARCHAR(20) UNIQUE, -- Quản lý nợ xấu qua CMND/CCCD [cite: 43]
    DiaChi NVARCHAR(200),
    NgayTao DATETIME DEFAULT GETDATE()
);

-- 2. Bảng Hợp đồng
CREATE TABLE HopDong (
    HopDongID INT IDENTITY(1,1) PRIMARY KEY,
    KhachHangID INT NOT NULL,
    SoTienVay DECIMAL(18,0) NOT NULL, -- Số tiền gốc [cite: 28]
    NgayVay DATE DEFAULT CAST(GETDATE() AS DATE),
    Deadline1 DATE NOT NULL, -- Mốc tính lãi đơn/kép [cite: 28]
    Deadline2 DATE NOT NULL, -- Mốc thanh lý [cite: 28]
    TrangThai NVARCHAR(50) DEFAULT N'Đang vay', -- [cite: 13]
    GhiChu NVARCHAR(500),
    CONSTRAINT FK_HopDong_KhachHang FOREIGN KEY (KhachHangID) REFERENCES KhachHang(KhachHangID)
);

-- 3. Bảng Danh mục Tài sản
CREATE TABLE TaiSan (
    TaiSanID INT IDENTITY(1,1) PRIMARY KEY,
    TenTaiSan NVARCHAR(200) NOT NULL,
    MoTa NVARCHAR(500),
    TrangThai NVARCHAR(50) DEFAULT N'Đang cầm cố' -- [cite: 49]
);

-- 4. Bảng Chi tiết Tài sản trong Hợp đồng (Giải quyết 1 hợp đồng nhiều tài sản) [cite: 9, 25]
CREATE TABLE HopDong_TaiSan (
    HopDongID INT NOT NULL,
    TaiSanID INT NOT NULL,
    GiaTriDinhGia DECIMAL(18,0) NOT NULL, -- [cite: 28]
    PRIMARY KEY (HopDongID, TaiSanID),
    CONSTRAINT FK_HDTS_HopDong FOREIGN KEY (HopDongID) REFERENCES HopDong(HopDongID),
    CONSTRAINT FK_HDTS_TaiSan FOREIGN KEY (TaiSanID) REFERENCES TaiSan(TaiSanID)
);

-- 5. Bảng Nhật ký giao dịch (Log)
CREATE TABLE LichSuGiaoDich (
    GiaoDichID INT IDENTITY(1,1) PRIMARY KEY,
    HopDongID INT NOT NULL,
    NgayGiaoDich DATETIME DEFAULT GETDATE(),
    SoTienTra DECIMAL(18,0) NOT NULL, -- [cite: 51]
    SoTienConNo DECIMAL(18,0), -- Tránh ghi đè gây mất dấu vết [cite: 52]
    LoaiGiaoDich NVARCHAR(50), -- Trả gốc, Trả lãi, Gia hạn... [cite: 50]
    NguoiThu NVARCHAR(100), -- [cite: 51]
    GhiChu NVARCHAR(500),
    CONSTRAINT FK_Log_HopDong FOREIGN KEY (HopDongID) REFERENCES HopDong(HopDongID)
);
GO
```

<img width="1918" height="1078" alt="image" src="https://github.com/user-attachments/assets/58612515-926a-42a1-b3ae-51fb5d6e6df4" />

## Event 1: Đăng ký hợp đồng mới
1. Khai báo kiểu dữ liệu danh sách tài sản
Việc này giúp bạn truyền nhiều món đồ vào cùng một lúc khi gọi Store Procedure.
```SQL
-- Tạo kiểu dữ liệu bảng để chứa danh sách tài sản thế chấp
CREATE TYPE AssetEntry AS TABLE (
    TenTaiSan NVARCHAR(200),
    MoTaTS NVARCHAR(500),
    GiaTriDinhGia DECIMAL(18,0)
);
GO
```

<img width="1918" height="1078" alt="image" src="https://github.com/user-attachments/assets/dec1f61c-fed4-4ecc-879b-46ef8665e242" />



2. Store Procedure tiếp nhận và in ra hợp đồng
```SQL
-- A. TẠO TYPE (Nếu bạn đã tạo rồi thì có thể bỏ qua phần này hoặc chạy lệnh DROP trước)
IF NOT EXISTS (SELECT * FROM sys.types WHERE name = 'AssetEntry')
BEGIN
    CREATE TYPE AssetEntry AS TABLE (
        TenTaiSan NVARCHAR(200),
        MoTaTS NVARCHAR(500),
        GiaTriDinhGia DECIMAL(18,0)
    );
END
GO

-- B. STORE PROCEDURE TIẾP NHẬN HỢP ĐỒNG
CREATE OR ALTER PROCEDURE sp_RegisterNewContract
    @HoTen NVARCHAR(100),
    @SoDienThoai VARCHAR(15),
    @CMND VARCHAR(20),
    @DiaChi NVARCHAR(200),
    @SoTienVayGoc DECIMAL(18,0),
    @Deadline1 DATE,
    @Deadline2 DATE,
    @DanhSachTaiSan AssetEntry READONLY
AS
BEGIN
    SET NOCOUNT ON;
    DECLARE @MaKH INT, @MaHD INT;

    BEGIN TRY
        BEGIN TRANSACTION;
        -- 1. Xử lý Khách hàng
        SELECT @MaKH = KhachHangID FROM KhachHang WHERE CMND = @CMND;
        IF @MaKH IS NULL
        BEGIN
            INSERT INTO KhachHang (HoTen, SoDienThoai, CMND, DiaChi)
            VALUES (@HoTen, @SoDienThoai, @CMND, @DiaChi);
            SET @MaKH = SCOPE_IDENTITY();
        END

        -- 2. Tạo Hợp đồng mới [cite: 28]
        INSERT INTO HopDong (KhachHangID, SoTienVay, NgayVay, Deadline1, Deadline2, TrangThai)
        VALUES (@MaKH, @SoTienVayGoc, GETDATE(), @Deadline1, @Deadline2, N'Đang vay');
        SET @MaHD = SCOPE_IDENTITY();

        -- 3. Xử lý danh sách Tài sản (Dùng Cursor để lấy từng dòng từ biến bảng)
        DECLARE @TenTS NVARCHAR(200), @MoTa NVARCHAR(500), @GiaTri DECIMAL(18,0);
        DECLARE @CurrentTSID INT;

        DECLARE db_cursor CURSOR FOR SELECT TenTaiSan, MoTaTS, GiaTriDinhGia FROM @DanhSachTaiSan;
        OPEN db_cursor;
        FETCH NEXT FROM db_cursor INTO @TenTS, @MoTa, @GiaTri;

        WHILE @@FETCH_STATUS = 0
        BEGIN
            -- Chèn vào bảng TaiSan
            INSERT INTO TaiSan (TenTaiSan, MoTa, TrangThai) VALUES (@TenTS, @MoTa, N'Đang cầm cố');
            SET @CurrentTSID = SCOPE_IDENTITY(); -- Lấy ID tài sản vừa tạo ngay lập tức

            -- Chèn vào bảng trung gian (Tránh lỗi Primary Key trùng ID)
            INSERT INTO HopDong_TaiSan (HopDongID, TaiSanID, GiaTriDinhGia)
            VALUES (@MaHD, @CurrentTSID, @GiaTri);

            FETCH NEXT FROM db_cursor INTO @TenTS, @MoTa, @GiaTri;
        END

        CLOSE db_cursor;
        DEALLOCATE db_cursor;

        COMMIT TRANSACTION;
        PRINT N'Đăng ký hợp đồng và tài sản thành công!';
    END TRY
    BEGIN CATCH
        IF @@TRANCOUNT > 0 ROLLBACK TRANSACTION;
        DECLARE @Err NVARCHAR(MAX) = ERROR_MESSAGE();
        RAISERROR(@Err, 16, 1);
    END CATCH
END;
GO

-- C. CHẠY THỬ NGHIỆM (EXECUTE)
DECLARE @TestAssets AssetEntry;
INSERT INTO @TestAssets VALUES 
(N'Xe Honda SH', N'Màu trắng, biển 20L1', 60000000),
(N'iPhone 15 Pro Max', N'Titan tự nhiên', 25000000);

EXEC sp_RegisterNewContract 
    @HoTen = N'Nguyễn Văn A', @SoDienThoai = '0912345678', @CMND = '012345678901', 
    @DiaChi = N'Thái Nguyên', @SoTienVayGoc = 40000000, 
    @Deadline1 = '2026-06-12', @Deadline2 = '2026-07-12', 
    @DanhSachTaiSan = @TestAssets;

-- KIỂM TRA DỮ LIỆU
SELECT * FROM KhachHang;
SELECT * FROM HopDong;
SELECT * FROM TaiSan;
SELECT * FROM HopDong_TaiSan;

```


<img width="1918" height="1078" alt="image" src="https://github.com/user-attachments/assets/2545b0ae-a275-4095-acc9-576fe10ce314" />

## Event 2: Tính toán công nợ thời gian thực.
- Function 1: fn_CalcMoneyTransaction
Hàm này tính số tiền của một giao dịch (hoặc biến động) cụ thể đến một ngày nhất định.  

```SQL
CREATE FUNCTION fn_CalcMoneyTransaction (@ContractID INT, @TargetDate DATE)
RETURNS DECIMAL(18,0)
AS
BEGIN
    DECLARE @SoTienVay DECIMAL(18,0);
    DECLARE @NgayVay DATE;
    DECLARE @D1 DATE;
    DECLARE @TongTien DECIMAL(18,0);
    DECLARE @LaiSuatNgay FLOAT = 0.005; -- 5.000 / 1.000.000 

    -- Lấy thông tin gốc từ hợp đồng [cite: 28]
    SELECT @SoTienVay = SoTienVay, @NgayVay = NgayVay, @D1 = Deadline1 
    FROM HopDong WHERE HopDongID = @ContractID;

    -- Nếu ngày tính toán trước ngày vay, tiền bằng 0
    IF @TargetDate < @NgayVay RETURN 0;

    -- TRƯỜNG HỢP 1: Tính lãi đơn (Từ ngày vay đến khi tới Deadline 1 hoặc TargetDate) 
    IF @TargetDate <= @D1
    BEGIN
        SET @TongTien = @SoTienVay + (@SoTienVay * @LaiSuatNgay * DATEDIFF(DAY, @NgayVay, @TargetDate));
    END
    -- TRƯỜNG HỢP 2: Tính lãi kép (Sau Deadline 1) [cite: 12, 32]
    ELSE
    BEGIN
        -- Tính tổng nợ tại mốc Deadline 1 (Gốc + Lãi đơn tích lũy) [cite: 12]
        DECLARE @NoTaiD1 DECIMAL(18,0) = @SoTienVay + (@SoTienVay * @LaiSuatNgay * DATEDIFF(DAY, @NgayVay, @D1));
        DECLARE @SoNgayQuaHan INT = DATEDIFF(DAY, @D1, @TargetDate);
        
        -- Công thức lãi kép: A = P * (1 + r)^n 
        SET @TongTien = @NoTaiD1 * POWER(1 + @LaiSuatNgay, @SoNgayQuaHan);
    END

    RETURN @TongTien;
END;
GO
```
<img width="1918" height="1078" alt="image" src="https://github.com/user-attachments/assets/b6fa6adf-971e-4671-b9af-a166d5f2118e" />

2. Function 2: fn_CalcMoneyContract
Hàm này tính tổng số tiền khách phải trả (Gốc + Lãi đơn + Lãi kép) và trừ đi số tiền khách đã trả (lấy từ bảng Log).

```SQL
CREATE FUNCTION fn_CalcMoneyContract (@ContractID INT, @TargetDate DATE)
RETURNS DECIMAL(18,0)
AS
BEGIN
    DECLARE @TongNoThucTe DECIMAL(18,0);
    DECLARE @TongDaTra DECIMAL(18,0);

    -- 1. Tính tổng nợ phát sinh đến ngày TargetDate bằng hàm đã viết ở trên [cite: 31]
    SET @TongNoThucTe = dbo.fn_CalcMoneyTransaction(@ContractID, @TargetDate);

    -- 2. Tính tổng số tiền khách đã trả (lấy từ bảng Log/LichSuGiaoDich) [cite: 51, 52]
    SELECT @TongDaTra = ISNULL(SUM(SoTienTra), 0) 
    FROM LichSuGiaoDich 
    WHERE HopDongID = @ContractID AND NgayGiaoDich <= @TargetDate;

    -- 3. Số tiền còn lại phải trả [cite: 31]
    RETURN @TongNoThucTe - @TongDaTra;
END;
GO
```

<img width="1918" height="1078" alt="image" src="https://github.com/user-attachments/assets/0665aedb-3e2f-4c45-9514-b7c95ef0ab40" />

3. Cách chạy thử nghiệm để kiểm tra
Sau khi tạo xong 2 hàm trên, bạn hãy chạy lệnh này để xem số tiền nợ nhảy theo thời gian (giả sử hôm nay là 1 tháng sau ngày vay):

```SQL
-- Xem nợ hiện tại của Hợp đồng 1 tính đến ngày 20/06/2026
SELECT 
    HopDongID, 
    SoTienVay AS Goc,
    dbo.fn_CalcMoneyTransaction(HopDongID, '2026-06-20') AS TongNo_BaoGomLai,
    dbo.fn_CalcMoneyContract(HopDongID, '2026-06-20') AS SoTienCanTraHienTai
FROM HopDong WHERE HopDongID = 1;
```

<img width="1918" height="1077" alt="image" src="https://github.com/user-attachments/assets/267276ad-4f9c-4d09-8fd0-030a6849323d" />

## Event 3: Xử lý trả nợ và hoàn trả tài sản.
Store Procedure: sp_ProcessRepayment
Thủ tục này sẽ thực hiện các bước sau:
- Kiểm tra thanh lý: Nếu tài sản đã bán, từ chối thu tiền.  
- Tính nợ & Trừ tiền: Sử dụng hàm fn_CalcMoneyContract đã viết ở Event 2 để lấy tổng nợ và cập nhật số tiền khách trả.
- Cập nhật trạng thái: Chuyển sang "Đã thanh toán đủ" hoặc "Đang trả góp" tùy số tiền.
- Gợi ý trả đồ: Chỉ đưa ra danh sách tài sản có thể trả lại nếu Giá trị đồ còn lại >= Dư nợ còn lại.  

```SQL
CREATE OR ALTER PROCEDURE sp_ProcessRepayment
    @ContractID INT,
    @AmountPaid DECIMAL(18,0),
    @StaffName NVARCHAR(100)
AS
BEGIN
    SET NOCOUNT ON;
    DECLARE @TotalDebt DECIMAL(18,0);
    DECLARE @RemainingDebt DECIMAL(18,0);
    DECLARE @ContractStatus NVARCHAR(50);

    -- 1. Kiểm tra nếu hợp đồng đã bị thanh lý tài sản
    SELECT @ContractStatus = TrangThai FROM HopDong WHERE HopDongID = @ContractID;
    
    IF @ContractStatus = N'Đã thanh lý'
    BEGIN
        PRINT N'Thông báo: Tài sản đã bị thanh lý. Không thu tiền, không trả đồ.';
        RETURN;
    END

    -- 2. Tính tổng nợ hiện tại (Gốc + Lãi) bằng Function ở Event 2
    SET @TotalDebt = dbo.fn_CalcMoneyContract(@ContractID, GETDATE());
    SET @RemainingDebt = @TotalDebt - @AmountPaid;

    -- 3. Ghi nhận vào Nhật ký giao dịch (Log)
    INSERT INTO LichSuGiaoDich (HopDongID, NgayGiaoDich, SoTienTra, SoTienConNo, LoaiGiaoDich, NguoiThu)
    VALUES (@ContractID, GETDATE(), @AmountPaid, @RemainingDebt, N'Khách trả nợ', @StaffName);

    -- 4. Cập nhật trạng thái hợp đồng và tài sản
    IF @RemainingDebt <= 0
    BEGIN
        UPDATE HopDong SET TrangThai = N'Đã thanh toán đủ' WHERE HopDongID = @ContractID;
        
        -- Nếu trả hết tiền thì cập nhật trạng thái tài sản thành "Đã trả khách"
        UPDATE TaiSan SET TrangThai = N'Đã trả khách' 
        WHERE TaiSanID IN (SELECT TaiSanID FROM HopDong_TaiSan WHERE HopDongID = @ContractID);
        
        PRINT N'Kết quả: Hợp đồng đã tất toán. Đã trả lại toàn bộ tài sản.';
    END
    ELSE
    BEGIN
        UPDATE HopDong SET TrangThai = N'Đang trả góp' WHERE HopDongID = @ContractID;
        
        -- 5. Gợi ý trả tài sản dựa trên điều kiện: Giá trị đồ còn lại >= Dư nợ còn lại
        PRINT N'Số tiền còn nợ sau giao dịch: ' + CAST(@RemainingDebt AS NVARCHAR(20));
        PRINT N'Gợi ý các tài sản có thể trả lại ngay cho khách:';
        
        SELECT TS.TenTaiSan, HT.GiaTriDinhGia
        FROM TaiSan TS
        JOIN HopDong_TaiSan HT ON TS.TaiSanID = HT.TaiSanID
        WHERE HT.HopDongID = @ContractID 
          AND TS.TrangThai = N'Đang cầm cố'
          AND (
            (SELECT SUM(GiaTriDinhGia) FROM HopDong_TaiSan WHERE HopDongID = @ContractID AND TaiSanID IN 
                (SELECT TaiSanID FROM TaiSan WHERE TrangThai = N'Đang cầm cố')) 
            - HT.GiaTriDinhGia >= @RemainingDebt
          );
    END
END;
GO

```

Test kiểm tra kết quả:
```SQL
-- Giả sử khách trả 15 triệu cho hợp đồng số 1
EXEC sp_ProcessRepayment @ContractID = 1, @AmountPaid = 15000000, @StaffName = N'Kiên';

-- Xem lại nhật ký trả tiền
SELECT * FROM LichSuGiaoDich;
```

<img width="1918" height="1078" alt="image" src="https://github.com/user-attachments/assets/3c2ce079-cf46-4c7a-9289-485e1cbf5241" />

## Event 4: Truy vấn danh sách nợ xấu

Em sẽ thay đổi ngày kiểm tra lùi về quá khứ hoặc cộng thêm ngày vào GETDATE()

```SQL
SELECT 
    KH.HoTen, 
    KH.SoDienThoai, 
    HD.SoTienVay AS Goc,
    -- Giả sử hôm nay là 20/06/2026 để test nợ xấu
    DATEDIFF(DAY, HD.Deadline1, '2026-06-20') AS SoNgayQuaHan,
    dbo.fn_CalcMoneyContract(HD.HopDongID, '2026-06-20') AS NoHienTai,
    dbo.fn_CalcMoneyContract(HD.HopDongID, DATEADD(MONTH, 1, '2026-06-20')) AS NoSau1Thang
FROM HopDong HD
JOIN KhachHang KH ON HD.KhachHangID = KH.KhachHangID
-- Ép điều kiện giả định để kiểm tra logic
WHERE CAST('2026-06-20' AS DATE) > HD.Deadline1 
  AND HD.TrangThai <> N'Đã thanh toán đủ';
```


<img width="1918" height="1078" alt="image" src="https://github.com/user-attachments/assets/d25c16ac-a049-48d5-bb05-0eab1242d0c0" />


## Event 5 - Quản lý thanh lý tài sản.

- Trigger 1: Chuyển trạng thái Hợp đồng sang "Quá hạn (nợ xấu)"
Trigger này kích hoạt khi hợp đồng đã vượt quá Deadline 1.

```SQL
CREATE OR ALTER TRIGGER trg_AutoOverdueContract
ON HopDong
AFTER UPDATE, INSERT
AS
BEGIN
    -- Chuyển từ "Đang vay" hoặc "Đang trả góp" sang "Quá hạn (nợ xấu)" nếu quá Deadline 1
    UPDATE HopDong
    SET TrangThai = N'Quá hạn (nợ xấu)'
    FROM HopDong
    INNER JOIN inserted i ON HopDong.HopDongID = i.HopDongID
    WHERE GETDATE() > HopDong.Deadline1 
      AND HopDong.TrangThai IN (N'Đang vay', N'Đang trả góp');
END;
GO

```

<img width="1918" height="1078" alt="image" src="https://github.com/user-attachments/assets/3fec5a4a-dfad-43c5-a29a-d177a9dabf8d" />


- Trigger 2: Chuyển trạng thái Tài sản sang "Sẵn sàng thanh lý"
Kích hoạt khi Hợp đồng đang nợ xấu mà ngày hiện tại vượt quá Deadline 2.


```SQL
CREATE OR ALTER TRIGGER trg_AutoReadyToSell
ON HopDong
AFTER UPDATE
AS
BEGIN
    SET NOCOUNT ON;

    -- Nếu Hợp đồng quá Deadline 2, chuyển tài sản liên quan sang "Sẵn sàng thanh lý"
    UPDATE TaiSan
    SET TrangThai = N'Sẵn sàng thanh lý'
    WHERE TaiSanID IN (
        SELECT HTS.TaiSanID  -- Đã sửa: Lấy trực tiếp từ HTS
        FROM HopDong_TaiSan HTS
        JOIN HopDong HD ON HTS.HopDongID = HD.HopDongID
        INNER JOIN inserted i ON HD.HopDongID = i.HopDongID
        WHERE GETDATE() > HD.Deadline2 
          AND HD.TrangThai = N'Quá hạn (nợ xấu)'
    ) 
    AND TaiSan.TrangThai = N'Đang cầm cố';
END;
GO



```
<img width="1918" height="1078" alt="image" src="https://github.com/user-attachments/assets/cc9b8a1c-886e-4060-8535-e8ccefbfd5da" />


3. Trigger 3: Chuyển trạng thái Tài sản thành "Đã bán thanh lý"
Khi chủ tiệm cập nhật trạng thái Hợp đồng thành "Đã thanh lý", Trigger này sẽ tự động đánh dấu các tài sản là đã bán xong.

```SQL
CREATE OR ALTER TRIGGER trg_AutoFinalizeAsset
ON HopDong
AFTER UPDATE
AS
BEGIN
    -- Nếu trạng thái Hợp đồng chuyển sang "Đã thanh lý"
    IF EXISTS (SELECT 1 FROM inserted WHERE TrangThai = N'Đã thanh lý')
    BEGIN
        UPDATE TaiSan
        SET TrangThai = N'Đã bán thanh lý'
        WHERE TaiSanID IN (
            SELECT TaiSanID 
            FROM HopDong_TaiSan 
            WHERE HopDongID IN (SELECT HopDongID FROM inserted WHERE TrangThai = N'Đã thanh lý')
        );
    END
END;
GO

```






<img width="1918" height="1078" alt="image" src="https://github.com/user-attachments/assets/e026ea4b-8499-44f2-877b-8a6cfeb6bb4e" />



- Cách kiểm tra (Test) Event 5:
Vì hôm nay là 12/05/2026, chưa tới Deadline của hợp đồng bạn vừa tạo, nên để thấy Trigger hoạt động, bạn hãy thử chạy lệnh "ép" ngày Deadline về quá khứ như sau:

```SQL
-- 1. Thử ép Deadline 1 về hôm qua để xem Trigger 1 nhảy nợ xấu
UPDATE HopDong 
SET Deadline1 = DATEADD(DAY, -1, GETDATE()) 
WHERE HopDongID = 1;

-- Kiểm tra xem trạng thái đã tự chuyển sang 'Quá hạn (nợ xấu)' chưa
SELECT HopDongID, TrangThai FROM HopDong WHERE HopDongID = 1;

-- 2. Thử cập nhật trạng thái hợp đồng thành 'Đã thanh lý' để xem Trigger 3 nhảy tài sản
UPDATE HopDong 
SET TrangThai = N'Đã thanh lý' 
WHERE HopDongID = 1;

-- Kiểm tra xem tài sản đã chuyển sang 'Đã bán thanh lý' chưa
SELECT * FROM TaiSan WHERE TaiSanID IN (SELECT TaiSanID FROM HopDong_TaiSan WHERE HopDongID = 1);

```




<img width="1918" height="1078" alt="image" src="https://github.com/user-attachments/assets/67899e80-1fd3-49af-b264-46a504c77fd0" />
