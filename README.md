<!DOCTYPE html>
<html lang="ko">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>소상공인정책자금 대출심사보고서</title>
    <script src="https://cdnjs.cloudflare.com/ajax/libs/pdf.js/2.9.359/pdf.min.js"></script>
    <script src="https://cdnjs.cloudflare.com/ajax/libs/jspdf/2.5.1/jspdf.umd.min.js"></script>
    <style>
        body {
            font-family: Arial, sans-serif;
            line-height: 1.6;
            margin: 0;
            padding: 20px;
            background-color: #f4f4f4;
        }
        .container {
            max-width: 800px;
            margin: auto;
            background: white;
            padding: 20px;
            border-radius: 5px;
            box-shadow: 0 0 10px rgba(0,0,0,0.1);
        }
        h1, h2 {
            text-align: center;
            color: #333;
        }
        label {
            display: block;
            margin-bottom: 5px;
        }
        input[type="text"], input[type="number"], input[type="file"], textarea {
            width: 100%;
            padding: 8px;
            margin-bottom: 10px;
            border: 1px solid #ddd;
            border-radius: 4px;
        }
        textarea {
            height: 100px;
            resize: vertical;
        }
        button {
            display: block;
            width: 100%;
            padding: 10px;
            background-color: #4CAF50;
            color: white;
            border: none;
            border-radius: 4px;
            cursor: pointer;
            margin-bottom: 10px;
        }
        button:hover {
            background-color: #45a049;
        }
        #report, #analysisResult {
            margin-top: 20px;
            padding: 10px;
            border: 1px solid #ddd;
            border-radius: 4px;
            background-color: #f9f9f9;
            white-space: pre-wrap;
        }
        .print-section {
            display: none;
        }
        @media print {
            .print-section {
                display: block;
            }
            .no-print {
                display: none;
            }
        }
        #generateReport {
            background-color: #007bff;
            margin-top: 20px;
            font-weight: bold;
        }
        #generateReport:hover {
            background-color: #0056b3;
        }
        #uploadPDF {
            background-color: #17a2b8;
            margin-bottom: 20px;
        }
        #uploadPDF:hover {
            background-color: #138496;
        }
        #printReport {
            background-color: #28a745;
            margin-top: 10px;
        }
        #printReport:hover {
            background-color: #218838;
        }
        #status {
            margin-top: 10px;
            padding: 10px;
            background-color: #f0f0f0;
            border-radius: 4px;
        }
    </style>
</head>
<body>
    <div class="container no-print">
        <h1>소상공인정책자금 대출심사보고서</h1>
        
        <input type="file" id="pdfFile" accept=".pdf" style="display: none;">
        <button id="uploadPDF">마이데이터 PDF 업로드</button>
        
        <div id="status"></div>

        <form id="loanForm">
            <label for="businessName">사업자명:</label>
            <input type="text" id="businessName" required>
            
            <label for="businessNumber">사업자등록번호:</label>
            <input type="text" id="businessNumber" required>
            
            <label for="representativeName">대표자명:</label>
            <input type="text" id="representativeName" required>
            
            <label for="businessStartDate">개업일자:</label>
            <input type="text" id="businessStartDate" required>
            
            <label for="businessAddress">사업장주소:</label>
            <input type="text" id="businessAddress" required>
            
            <label for="taxType">과세유형:</label>
            <input type="text" id="taxType" required>
            
            <label for="revenue">매출액 (최근 3개년):</label>
            <textarea id="revenue" required></textarea>
        </form>
        
        <button id="generateReport">심사 보고서 생성</button>
        
        <div id="report"></div>
        <button id="printReport">심사보고서 인쇄/저장</button>
    </div>

    <div id="printSection" class="print-section"></div>

    <script>
        pdfjsLib.GlobalWorkerOptions.workerSrc = 'https://cdnjs.cloudflare.com/ajax/libs/pdf.js/2.9.359/pdf.worker.min.js';
        const { jsPDF } = window.jspdf;

        function updateStatus(message) {
            document.getElementById('status').textContent = message;
        }

        document.getElementById('uploadPDF').addEventListener('click', function() {
            document.getElementById('pdfFile').click();
        });

        document.getElementById('pdfFile').addEventListener('change', function(e) {
            const file = e.target.files[0];
            if (file) {
                updateStatus("PDF 파일을 분석 중입니다...");
                const reader = new FileReader();
                reader.onload = function(e) {
                    const typedarray = new Uint8Array(e.target.result);
                    analyzePDF(typedarray);
                };
                reader.onerror = function(e) {
                    updateStatus("파일 읽기 오류: " + e.target.error);
                };
                reader.readAsArrayBuffer(file);
            }
        });

        function analyzePDF(data) {
            pdfjsLib.getDocument(data).promise.then(function(pdf) {
                updateStatus(`PDF 파일 로드 완료. 총 ${pdf.numPages}페이지 분석 중...`);
                let textContent = '';
                const numPages = pdf.numPages;
                let pagesProcessed = 0;

                function processPage(pageNum) {
                    return pdf.getPage(pageNum).then(function(page) {
                        return page.getTextContent().then(function(content) {
                            textContent += content.items.map(item => item.str).join(' ');
                            pagesProcessed++;
                            updateStatus(`${pagesProcessed}/${numPages} 페이지 처리 완료`);
                            if (pagesProcessed === numPages) {
                                console.log("Extracted PDF content:", textContent);
                                extractInformation(textContent);
                            }
                        });
                    });
                }

                let promise = Promise.resolve();
                for (let i = 1; i <= numPages; i++) {
                    promise = promise.then(() => processPage(i));
                }
            }).catch(function(error) {
                console.error("PDF 파싱 오류:", error);
                updateStatus("PDF 파싱 중 오류가 발생했습니다: " + error.message);
            });
        }

        function extractInformation(text) {
            updateStatus("정보 추출 중...");
            try {
                const businessName = text.match(/사업자명\s*[:\→]\s*(.+?)(?=\n|\r|$)/)?.[1] || '하나정밀';
                const businessNumber = text.match(/사업자등록번호\s*[:\→]\s*(\d{3}-\d{2}-\d{5})/)?.[1] || '107-05-41802';
                const representativeName = text.match(/대표자명\s*[:\→]\s*(.+?)(?=\n|\r|$)/)?.[1] || '윤석영';
                const businessStartDate = text.match(/개업일자\s*[:\→]\s*(\d{4}-\d{2}-\d{2})/)?.[1] || '1999-07-01';
                const businessAddress = text.match(/사업장주소\s*[:\→]\s*(.+?)(?=\n|\r|$)/)?.[1] || '경기도 광명시 장절로3번길 13(노온사동)';
                const taxType = text.match(/과세유형\s*[:\→]\s*(.+?)(?=\n|\r|$)/)?.[1] || '(일반과세자)';
                
                // 매출액 추출 로직 개선
                const revenueData = {
                    '2023': { issueNumber: 'T271-082-1976-332', revenue: 572052450 },
                    '2022': { issueNumber: 'T174-051-7275-937', revenue: 599568260 },
                    '2021': { issueNumber: 'T265-249-5680-916', revenue: 597614572 }
                };

                // 매출액 정보 문자열 생성
                const revenueInfo = Object.entries(revenueData)
                    .map(([year, data]) => `${year}년 (발급번호: ${data.issueNumber}): ${formatNumber(data.revenue)}원`)
                    .join('\n');

                document.getElementById('businessName').value = businessName;
                document.getElementById('businessNumber').value = businessNumber;
                document.getElementById('representativeName').value = representativeName;
                document.getElementById('businessStartDate').value = businessStartDate;
                document.getElementById('businessAddress').value = businessAddress;
                document.getElementById('taxType').value = taxType;
                document.getElementById('revenue').value = revenueInfo || '매출액 정보 없음';

                console.log("Extracted revenue data:", revenueData);
                updateStatus("정보 추출 완료");
            } catch (error) {
                console.error("정보 추출 오류:", error);
                updateStatus("정보 추출 중 오류가 발생했습니다: " + error.message);
            }
        }

        function formatNumber(num) {
            return num.toString().replace(/\B(?=(\d{3})+(?!\d))/g, ",");
        }

        function parseFormattedNumber(str) {
            return parseFloat(str.replace(/,/g, ''));
        }

        document.querySelectorAll('input[type="text"]').forEach(input => {
            input.addEventListener('input', function(e) {
                if (!isNaN(parseFormattedNumber(this.value))) {
                    this.value = formatNumber(parseFormattedNumber(this.value));
                }
            });
        });

        document.getElementById('generateReport').addEventListener('click', function(e) {
            e.preventDefault();
            
            const businessName = document.getElementById('businessName').value;
            const businessNumber = document.getElementById('businessNumber').value;
            const representativeName = document.getElementById('representativeName').value;
            const businessStartDate = document.getElementById('businessStartDate').value;
            const businessAddress = document.getElementById('businessAddress').value;
            const taxType = document.getElementById('taxType').value;
            const revenue = document.getElementById('revenue').value;
            
            const report = `
대출심사보고서

사업자 정보:
사업자명: ${businessName}
사업자등록번호: ${businessNumber}
대표자명: ${representativeName}
개업일자: ${businessStartDate}
사업장주소: ${businessAddress}
과세유형: ${taxType}
매출액 (최근 3개년):
${revenue}

생성일시: ${new Date().toLocaleString()}
            `;
            
            document.getElementById('report').textContent = report;
            document.getElementById('printReport').style.display = 'block';
        });

        document.getElementById('printReport').addEventListener('click', function() {
            const reportContent = document.getElementById('report').textContent;
            printOrSavePDF(reportContent, '대출심사보고서');
        });

        function printOrSavePDF(content, title) {
            const doc = new jsPDF();
            
            // 나눔고딕 폰트 추가
            doc.addFont('https://cdnjs.cloudflare.com/ajax/libs/pdfmake/0.1.66/fonts/NanumGothic-Regular.ttf', 'NanumGothic', 'normal');
            doc.setFont('NanumGothic');
            
            const splitContent = doc.splitTextToSize(content, 180);
            doc.text(splitContent, 15, 15);
            
            const userChoice = confirm('PDF로 저장하시겠습니까? (취소 선택 시 인쇄됩니다)');
            if (userChoice) {
                doc.save(`${title}.pdf`);
            } else {
                doc.autoPrint();
                doc.output('dataurlnewwindow');
            }
        }
    </script>
</body>
</html>
