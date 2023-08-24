### 목차

1. [문제상황](#문제상황)
2. [원인](#원인)
3. [해결](#해결)


### 문제상황

![image](https://github.com/shinyubin989/note/assets/69676101/af3d2f57-69a9-42d9-8992-5aca099da4ae)


- 회원가입을 하는 로직에서, 만약 새 clientId의 회원가입 요청이라면 save로직을 수행하고, 없다면 수행하지 않도록 했음
- 이때 findByClientId()는 Optional 반환값을 가지고, Optional이 Empty일때만 save로직을 수행하는 것을 의도함

![image](https://github.com/shinyubin989/note/assets/69676101/aebff077-2b6d-4456-affa-63737b46cc24)


- 테스트 로직은 위와 같았는데, 테스트가 계속 실패하는 문제 상황을 맞닥뜨림

### 원인

1. Optional의 orElse()는 Optional에 값이 있든 없든 무조건 실행됨. 그러나 orElseGet(Supplier)의 Supplier는 Optional에 값이 없을 때만 실행됨. 따라서 Optional에 값이 없을 때만 새 객체를 생성하거나 새 연산을 수행하므로 불필요한 오버헤드가 없음
    
    ![image](https://github.com/shinyubin989/note/assets/69676101/77feab8f-eb2d-4686-b7e5-aff2bab748f0)

    
2. 값을 인자로서 넘겨줄 때 이미 save로직을 수행해야하므로 연산 순서가 “save로직 수행 → orElse(~)에 인자로 넘김”와 같이 됨. 따라서 verifyNoMoreInteractions() 테스트에 실패함

### 해결

- orElse()를 orElseGet()으로 바꿔 해결함
