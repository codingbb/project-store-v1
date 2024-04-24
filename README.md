# final project 1단계 - 상품 판매 사이트

<hr>

## [ 1단계 기능 구현 ]
* 메인 페이지 및 상품 등록, 수정, 삭제, 상세보기, 목록보기
* *현재 ResponseDTO는 사용하지 않음. Entity 사용
* 기술 스택 : Springboot 3.2, JDK 21, IntelliJ, MySQL 8.0
* 의존성 : Lombok, Spring Boot DevTools, Spring Web, Spring Data JPA, MySQL Driver, Mustache 

<hr>

## 1. MySQL 연동
* username : root / password : 1234
![image](https://github.com/codingbb/project-store-v1/assets/153585866/e5484b56-6230-41b3-bf61-ab657d1f3c46)

* 사용자 생성 및 권한 주기, DB 생성

![image](https://github.com/codingbb/project-store-v1/assets/153585866/56aa2b23-5047-41df-9762-c3c040efde06)

<hr>

## 2. yml 설정
```yml
server:
  servlet:
    encoding:
      charset: utf-8
      force: true
  port: 8080

spring:
  mustache:
    servlet:
      expose-session-attributes: true
      expose-request-attributes: true
  datasource:
    driver-class-name: com.mysql.cj.jdbc.Driver
    url: jdbc:mysql://localhost:3306/store?serverTimezone=UTC&characterEncoding=utf-8
    username: root
    password: 1234
  h2:
    console:
      enabled: true
  sql:
    init:
      data-locations:
#        - classpath:db/data.sql
  jpa:
    hibernate:
      ddl-auto: update
    show-sql: true
    properties:
      hibernate:
        format_sql: true
    defer-datasource-initialization: true
    open-in-view: false
```

<hr>

## 3. build.gradle -> MySQL 의존성 추가
```
implementation 'mysql:mysql-connector-java:8.0.28'
```
<hr>

## 4. Entity 생성 - Product 
```java
package com.example.store_v1.product;

import jakarta.persistence.*;
import lombok.Builder;
import lombok.Data;
import lombok.NoArgsConstructor;
import org.hibernate.annotations.CreationTimestamp;
import java.time.LocalDateTime;

@NoArgsConstructor
@Data
@Table(name = "product_tb")
@Entity
public class Product {
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Integer id;

    @Column(unique = true, length = 20, nullable = false)
    private String name;

    @Column(nullable = false)
    private Integer price;
    //재고
    @Column(nullable = false)
    private Integer qty;

    //이미지용 //파일 이름(파일 경로)
    private String imgFileName;

    @CreationTimestamp
    private LocalDateTime createdAt;

    @Transient
    private Integer indexNumb;

    @Builder
    public Product(Integer id, String name, Integer price, Integer qty, LocalDateTime createdAt) {
        this.id = id;
        this.name = name;
        this.price = price;
        this.qty = qty;
        this.createdAt = createdAt;
    }
}

```
* 추후 이미지를 받기 위해 imgFileName 필드 생성

<hr>

## 5. index 페이지

### 5-1. ProductController - main
```java
    @GetMapping("/")
    public String main(HttpServletRequest request) {
        List<ProductResponse.MainDTO> productList = productService.findAllMain();
        System.out.println(productList);
        request.setAttribute("productList", productList);
        return "/index";
    }
```

### 5-2. ProductService - findAllMain
```java
    //상품 메인 목록 보기
    public List<ProductResponse.MainDTO> findAllMain() {
        List<Product> productList = productRepo.findAll();

        //엔티티 받아온걸 dto로 변경
        return productList.stream().map(product ->
                new ProductResponse.MainDTO(product)).collect(Collectors.toList());
    }
```

### 5-3. ProductRepoitory - findAll
```java
    //상품 목록보기
    public List<Product> findAll() {
        String q = """
                select * from product_tb order by id desc 
                """;
        Query query = em.createNativeQuery(q, Product.class);
        List<Product> productList = query.getResultList();
        return productList;
    }
```

<hr>

## 6. 상품 등록하기 - save
### 6-1. ProductController
```java
// 상품 등록
    @PostMapping("/product/save")
    public String save(ProductRequest.SaveDTO requestDTO) {
        productService.save(requestDTO);

        return "redirect:/product";
    }


    @GetMapping("/product/save-form")
    public String saveForm() {

        return "/product/save-form";
    }
```

### 6-2. ProductService
```java
    //상품 등록
    @Transactional
    public void save(ProductRequest.SaveDTO requestDTO) {
        productRepo.save(requestDTO);
    }
```

### 6-3. ProductRepository
```java
//상품 등록
    public void save(ProductRequest.SaveDTO requestDTO) {
        String q = """
                insert into product_tb(name, price, qty, created_at) values (?, ?, ?, now())
                """;

        Query query = em.createNativeQuery(q);
        query.setParameter(1, requestDTO.getName());
        query.setParameter(2, requestDTO.getPrice());
        query.setParameter(3, requestDTO.getQty());
        query.executeUpdate();
    }
```

### 6-4. ProductRequest - SaveDTO
```java
    @Data
    public static class SaveDTO {
        private String name;
        private Integer price;
        private Integer qty;

    }
```

<hr>

## 7. 상품 수정하기 update
### 7-1. ProductController
```java
//업데이트 폼 (업데이트는 2개가 나와야함)
    @GetMapping("/product/{id}/update-form")
    public String updateForm(@PathVariable Integer id, HttpServletRequest request) {
        Product product = productService.findByIdUpdate(id);
        request.setAttribute("product", product);
        return "/product/update-form";
    }

    //업데이트
    @PostMapping("/product/{id}/update")
    public String update(@PathVariable Integer id, ProductRequest.UpdateDTO requestDTO) {
        productService.updateById(id, requestDTO);
        return "redirect:/product";
    }
```

### 7-2. ProductService
```java
    //상품 업데이트 폼 보기
    public Product findByIdUpdate(Integer id) {
        Product product = productRepo.findById(id);
        return product;
    }
    // 상품 업데이트
    @Transactional
    public void updateById(Integer id, ProductRequest.UpdateDTO requestDTO) {
        productRepo.updateById(id, requestDTO);

    }
```

### 7-3. ProductRepository
```java
    //상품 수정하기
    public void updateById(Integer id, ProductRequest.UpdateDTO requestDTO) {
        String q = """
                update product_tb set name = ?, price = ?, qty = ? where id = ?
                """;
        Query query = em.createNativeQuery(q);
        query.setParameter(1, requestDTO.getName());
        query.setParameter(2, requestDTO.getPrice());
        query.setParameter(3, requestDTO.getQty());
        query.setParameter(4, id);
        query.executeUpdate();

    }
```

### 7-4. ProductRequest - UpdateDTO
```java
    @Data
    public static class UpdateDTO {
        private String name;
        private Integer price;
        private Integer qty;
    }
```

<hr>

## 8. 상품 목록보기
### 8-1. ProductController
```java
//상품 목록보기
    @GetMapping("/product")
    public String listForm(HttpServletRequest request) {
        List<Product> productList = productService.findAllList();
        request.setAttribute("productList", productList);

        return "/product/list";
    }
```
### 8-2. ProductService
```java
//상품 리스트 목록 보기
    public List<Product> findAllList() {
        List<Product> productList = productRepo.findAll();

        //pk로 no 주니까 너무 지저분해져서 no용 필드를 새로 만들어줌
        Integer indexNumb = productList.size();
        for (Product product : productList) {
            product.setIndexNumb(indexNumb--);
        }

        //엔티티 받아온걸 dto로 변경
        return productList;
    }
```
* 화면에 있는 [ No ] 표현 로직
  
### 8-3. ProductRepository
```java
    //상품 목록보기
    public List<Product> findAll() {
        String q = """
                select * from product_tb order by id desc 
                """;
        Query query = em.createNativeQuery(q, Product.class);
        List<Product> productList = query.getResultList();
        return productList;
    }
```
* 재활용 

<hr>

## 9. 상품 상세보기 
### 9-1. ProductController
```java
    //상품 상세보기
    @GetMapping("/product/{id}")
    public String detail(@PathVariable Integer id, HttpServletRequest request) {
        Product product = productService.findByIdDetail(id);
        request.setAttribute("product", product);

        System.out.println(product);
        return "/product/detail";
    }
```

### 9-2. ProductService
```java
    //상품 상세보기
    public Product findByIdDetail(Integer id) {
        Product product = productRepo.findById(id);
        return product;
    }
```

### 9-3. ProductRepository
```java
    //상품 상세보기
    public Product findById(Integer id) {
        String q = """
                select * from product_tb where id = ?
                """;
        Query query = em.createNativeQuery(q, Product.class);
        query.setParameter(1, id);
        Product result = (Product) query.getSingleResult();
        return result;
    }
```
* 재활용 

<hr>

## 10. 상품 삭제하기
### 10-1. ProductController
```java
    //상품 삭제
    @PostMapping("/product/{id}/delete")
    public String deleteById(@PathVariable Integer id) {
        productService.deleteById(id);
        //product/list를 반환하는 컨트롤러 존재 -> redirect 할 것
        //어려우면 PostMapping은 redirect로 준다고 생각하세요
        return "redirect:/product";
    }
```

### 10-2. ProductService
```java
    //상품 삭제하기
    @Transactional
    public void deleteById(Integer id) {
        //추후 delete 할 때, 존재하는지 확인하고 들어갈 것. 지금은 생략
        productRepo.deleteById(id);
    }
```

### 10-3. ProductRepository
```java
    //상품 삭제하기
    public void deleteById(Integer id) {
        String q = """
                delete from product_tb where id = ?
                """;
        Query query = em.createNativeQuery(q);
        query.setParameter(1, id);
        query.executeUpdate();

    }
```


