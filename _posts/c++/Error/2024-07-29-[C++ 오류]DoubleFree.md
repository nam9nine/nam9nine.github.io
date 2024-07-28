---
title: "[C++ Error] Double Free 에러 해결하기"
date: 2024-07-29 00:41:00 +0900
categories: [C++, Error]
tags: [C++]
author: me
---
# Double Free 에러 해결하기

## 문제 발생

`MyString` 클래스의 객체를 복사하는 과정에서 발생한 double free 오류입니다.

```cpp
// MyString.cpp
class MyString {
public:
    MyString(const char* init) {
        size_ = strlen(init);
        str_ = new char[size_ + 1];
        strcpy(str_, init);
    }
    MyString(const MyString& copy) {
        str_ = copy.str_;  // 문제의 원인: 얕은 복사
        size_ = copy.size_;
    }
    ~MyString() {
        delete[] str_;  // 여기에서 double free 발생
    }

private:
    char* str_;
    int size_;
};

// main.cpp
int main() {
    MyString str1("Hello World!");
    MyString str2 = str1;  // 복사 생성자 사용
    return 0;  // 두 객체의 소멸 시 double free 발생
}
```

오류 메시지:
```shell
main(57049,0x1fad68c00) malloc: Double free of object 0x14ba040b0
main(57049,0x1fad68c00) malloc: *** set a breakpoint in malloc_error_break to debug
```

## 문제 분석

### 오류 메세지에서의 힌트

메모리 주소 `0x14ba040b0`에서 객체의 메모리가 두 번 해제되었습니다. `str1`과 `str2`는 동일한 메모리 주소를 가리키는 `str_`를 공유하므로, 한 객체의 소멸자가 메모리를 해제한 후 또 다른 객체의 소멸자가 동일한 메모리 주소를 해제하려고 시도했습니다.

### 왜 같은 객체가 2번 해제되는가?

`MyString`의 복사 생성자가 멤버 변수 `str_`에 대해 깊은 복사(deep copy)를 수행하지 않고 얕은 복사(shallow copy)를 수행했기 때문입니다. 이로 인해 두 객체가 메모리 주소를 공유하며, 한 객체가 소멸하면서 할당된 메모리를 해제하고, 다른 객체가 다시 같은 메모리를 해제하려고 하여 double free 오류가 발생합니다.

### 왜 2번 해제되면 안 되는가?

메모리가 한 번 해제되면, 해당 메모리 주소는 시스템에 의해 재사용 준비가 됩니다. 이미 해제된 메모리 주소를 다시 해제하려고 하면 메모리 관리 시스템이 손상될 수 있으며, 이는 프로그램의 충돌이나 예측 불가능한 다른 행동을 초래할 수 있습니다. 또한, 보안 측면에서도 큰 위험을 수반합니다.

## 문제 해결

이 문제를 해결하기 위해 복사 생성자를 수정하여 각 객체가 자신의 메모리를 독립적으로 가질 수 있도록 깊은 복사를 구현합니다.

```cpp
MyString(const MyString& copy) {
    size_ = copy.size_;
    str_ = new char[size_ + 1];
    strcpy(str_, copy.str_);  // 깊은 복사 수행
}
```

이 수정으로 각 `MyString` 객체는 별도의 메모리 영역을 가지게 되어 소멸 시점에 발생하는 문제를 방지할 수 있습니다.

## 결론

Double Free는 메모리 관리에 있어서 심각한 오류로, 적절한 복사 생성자의 구현이 필수적입니다. 프로그램의 안정성을 확보하고 보안 문제를 예방하기 위해 깊은 복사를 통한 메모리 관리가 중요합니다.


