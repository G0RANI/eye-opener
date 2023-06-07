## querydsl을 사용하는 대용량 엑셀 다운로드시 OutOfMemory 를 개선해보자

## 1. 개요

회사의 비지니스 특성상 월말 정산 작업이 있는데 엑셀을 수기로 다운 받아 처리하는 업무가 많다 (개선을 해준다 해도 자신들의 눈과 손을 더 믿는듯 하다...)

따라서 월말에 대량의 엑셀 다운로드가 이루어 지고 그에 따라 OutOfMemory의 메세지를 받는 경우가 많았다.

일단 조회 범위를 줄이는 방법으로 선 조치 후 해당 내용에 대하여 개선한 내용을 남긴다.



## 2. 문제점

말 그대로 OutOfMemory라는 메세지를 받았으니 엑셀 다운로드 할때 어느 부분이 메모리를 잡아 먹는지 알아봤다.

크게 두가지의 문제점이 있었다.

### 2.1 조회

```java
List<Dto> resultList = getQuerydsl().....from(table).fetch()
```

- 대량 데이터를 조회하고 다운로드할 때 다운로드 대상 전체를 메모리에 모두 적체 후 처리하고 있었다.
- Heap Size 를 늘려도 동시 요청 때 동일 증상이 계속 발생할것이다.

### 2.2 다운로드 진행

```java
public <T> InputStreamResource exportExcel(List<T> exportList, Class<T> exportClass) {
  try (ByteArrayOutputStream out = new ByteArrayOutputStream();
      SXSSFWorkbook workbook = ExcelWriter.builder()
        .type(exportClass)
        .write(exportList)) {
    workbook.write(out);
    workbook.dispose();
    return new InputStreamResource(new ByteArrayInputStream(out.toByteArray()));
  } catch (IOException e) {
    throw new ExportExcelException();
  }
}
```

- 엑셀 파일을 생성 후 메모리에 모두 적제 하고 다운을 진행하고 있다.



## 3. 해결

### 3.1 조회

조회는 QueryDsl을 사용하고 있고 조회한 데이터를 메모리에 안올릴수 있는 방법을 찾아보았다.

크게 2가지 방법 정도를 찾았다.



#### `stream()` 메서드를 사용하는 방법
- 효율성: 데이터를 필요한 만큼 처리하므로 일부 데이터만 사용하는 경우에 유리.
- 적합한 상황: 작은 규모의 데이터나 일부 데이터만 처리해야 할 때, 코드의 가독성과 유연성을 강조하는 경우에 사용.

```java
Stream<Dto> resultList = getQuerydsl().....from(table).stream()
```

Java 8의 Stream API와 유사한 방식으로 데이터를 스트림으로 처리하는 방식이다.

이를 통해 데이터를 메모리에 한 번에 로드하지 않고, 필요한 만큼 데이터를 스트림으로 처리하면서 메모리 사용을 최적화 할 수 있다. 

스트림은 데이터를 순차적으로 처리하며, 중간 연산과 최종 연산을 통해 데이터를 가공하고 결과를 얻을 수 있다고 한다.




#### `iterate()` 메서드를 사용하는 방법
- 효율성: 대용량 데이터 처리에 특화되어 있으며, 메모리 사용량을 최소화.
- 적합한 상황: 대량의 데이터를 효율적으로 처리해야 할 때, 메모리 사용량을 관리하고 데이터베이스 연결을 효율적으로 관리해야 할 때 사용.

```java
CloseableIterator<Dto> resultList = getQuerydsl().....from(table).iterate()
```

`iterate()` 메서드를 사용하여 `CloseableIterator`를 받는 경우, 데이터를 반복적으로 조회하면서 필요한 만큼 데이터를 로드한다. 

`iterate()` 메서드를 사용하면 데이터를 한 번에 메모리에 올리는 것이 아니라, 필요한 만큼 데이터를 가져와서 처리. 



2가지 방법중 대용량에 초점을 맞추어  `iterate()` 메서드를 사용하여 `CloseableIterator`를 받는 방법을 선택하였다.




### 3.2 다운로드 진행

```java
Workbook workbook = new XSSFWorkbook();
Sheet sheet = workbook.createSheet("Data");

// 헤더 생성
Row headerRow = sheet.createRow(0);
headerRow.createCell(0).setCellValue("Column 1");
headerRow.createCell(1).setCellValue("Column 2");
// ...

// 데이터 삽입
int rowIndex = 1;
while (resultList.hasNext()) {
    Entity entity = resultList.next();
    Row row = sheet.createRow(rowIndex++);
    row.createCell(0).setCellValue(entity.getColumn1());
    row.createCell(1).setCellValue(entity.getColumn2());
    // ...
}

// Excel 파일을 스트림으로 전송하는 StreamingOutput 구현
StreamingOutput streamingOutput = outputStream -> {
    try {
        workbook.write(outputStream);
        outputStream.flush();
    } finally {
        workbook.close();
        resultList.close(); // CloseableIterator 닫기
    }
};

// 파일 다운로드 설정
response.setContentType("application/vnd.openxmlformats-officedocument.spreadsheetml.sheet");
response.setHeader("Content-Disposition", "attachment; filename=\"data.xlsx\"");

// StreamingOutput을 사용하여 데이터를 클라이언트로 전송
try {
    streamingOutput.write(response.getOutputStream());
} catch (IOException e) {
    e.printStackTrace();
}
```

`StreamingOutput`을 구현하여 Excel 파일을 스트림으로 전송하였다. 

`StreamingOutput`의 `write` 메서드 내에서 Excel 파일을 클라이언트로 전송한 후, 스트림과 워크북을 정리한다.

이 방법을 사용하면 Excel 파일을 메모리에 올리지 않고 바로 다운로드할 수 있다.

주의 해야 할 점은 `CloseableIterator`는 반드시 `iterator.close()`를 호출하여 닫아주어야 한다. 

`StreamingOutput`에서 `workbook.close()`와 함께 `iterator.close()`를 호출하여 자원을 해제해야 한다.

그렇지 않으면 Connection을 제대로 닫지 않거나 예외가 발생하여 롤백되지 않는 경우 Connection Leak이 발생할 수 있다.

`iterator.close()`를 호출하지 않고 `@Transactional` 애노테이션을 사용하여 자원을 관리 하는 방법이 있다.



## 4. 후기

해당 방법을 사용하여 개선 후 그리고 새로 적용해 나아간 내용에 대해서는 문제가 발생하지 않았다.

이 내용을 팀원에게 공유를 하였지만 어렵다...? 자신의 능력 밖이다 란 말과 함께 적용은 생각해보겠다...라고 한다..

후...아쉬워라 함께 생각하고 공유할 수 있는 동료가 생겼으면 좋겠다.

