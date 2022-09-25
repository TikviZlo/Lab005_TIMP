[![Coverage Status](https://coveralls.io/repos/github/TikviZlo/Lab005_TIMP/badge.svg?branch=master)](https://coveralls.io/github/TikviZlo/Lab005_TIMP?branch=master)
## 1. В коде Transactions.cpp была ошибка

В 39 строчке программы стояла to, а не from из-за чего изначально летали все тесты.

## 2. Первый шаг: добавить все файлы banking и написать CMakeLists.txt:

```
cmake_minimum_required(VERSION 3.18)

set(CMAKE_CXX_STANDARD 20)

option(BUILD_TESTS "Build tests" OFF)
option(COVERALLS "Check coverage" OFF)

project(bank)

add_library(account STATIC Account.cpp Account.h)
add_library(transaction STATIC Transaction.cpp Transaction.h)
include_directories(${CMAKE_CURRENT_SOURCE_DIR})

include(FetchContent)
FetchContent_Declare(
        googletest
        URL https://github.com/google/googletest/archive/609281088cfefc76f9d0ce82e1ff6c30cc3591e5.zip
)
set(gtest_force_shared_crt ON CACHE BOOL "" FORCE)
FetchContent_MakeAvailable(googletest)

if (BUILD_TESTS)
    include(GoogleTest)
    enable_testing()
    add_executable(bank_test ${CMAKE_CURRENT_SOURCE_DIR}/../test.cpp)
    target_link_libraries(bank_test account transaction gtest_main gmock_main)
    gtest_discover_tests(bank_test)
endif ()

if (COVERALLS)
    SET(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -fprofile-arcs -ftest-coverage")
    SET(CMAKE_C_FLAGS "${CMAKE_C_FLAGS} -fprofile-arcs -ftest-coverage")
    SET(CMAKE_EXE_LINKER_FLAGS "${CMAKE_EXE_LINKER_FLAGS} -fprofile-arcs -ftest-coverage")
endif ()
```

## 3. Написать тесты в test.cpp:

```
#include <Account.h>
#include <Transaction.h>
#include <gmock/gmock.h>
#include <gtest/gtest.h>

using ::testing::Return;

class MockAccount : public Account {
public:
    MockAccount(int id, int balance) : Account(id, balance) {}

    MOCK_METHOD(int, GetBalance, (), (const, override));
    MOCK_METHOD(void, ChangeBalance, (int diff), (override));
    MOCK_METHOD(void, Lock, (), (override));
    MOCK_METHOD(void, Unlock, (), (override));
};

TEST(BankTest, Bank) {
    MockAccount t1(1, 220);
    MockAccount t2(2, 100);
    Transaction test;
    EXPECT_CALL(t1, GetBalance())
        .WillOnce(testing::Return(220))
        .WillOnce(testing::Return(69))
        .WillOnce(testing::Return(69))
        .WillOnce(testing::Return(69));
    EXPECT_CALL(t2, GetBalance())
        .WillOnce(testing::Return(250))
        .WillOnce(testing::Return(250));
    EXPECT_CALL(t1, ChangeBalance(-151))
        .WillOnce(testing::Invoke(nullptr));
    EXPECT_CALL(t2, ChangeBalance(testing::_))
        .Times(3);
    EXPECT_CALL(t1, Lock())
        .Times(5)
        .WillOnce(testing::Throw(std::runtime_error("at first lock the account")))
        .WillOnce(nullptr)
        .WillOnce(testing::Throw(std::runtime_error("already locked")))
        .WillOnce(nullptr)
        .WillOnce(nullptr);
    EXPECT_CALL(t2, Lock())
        .Times(2);
    EXPECT_CALL(t1, Unlock())
        .Times(3);
    EXPECT_CALL(t2, Unlock())
        .Times(2);

    EXPECT_THROW(test.Make(t1, t1, 50), std::logic_error);
    EXPECT_THROW(test.Make(t1, t2, 50), std::logic_error);
    EXPECT_THROW(test.Make(t1, t2, -50), std::invalid_argument);

    test.set_fee(200);
    EXPECT_EQ(test.Make(t1, t2, 150), 0);
    EXPECT_EQ(test.fee(), 200);
    test.set_fee(1);

    EXPECT_THROW(test.Make(t1, t2, 150), std::runtime_error);
    t1.Lock();
    EXPECT_THROW(test.Make(t1, t2, 150), std::runtime_error);
    t1.Unlock();

    bool res = test.Make(t1, t2, 150);
    EXPECT_EQ(res, 1);

    res = test.Make(t1, t2, 150);
    EXPECT_EQ(res, 0);
}

TEST(BankTest, Account) {
    Account t1(1, 500);

    EXPECT_EQ(t1.GetBalance(), 500);
    EXPECT_THROW(t1.ChangeBalance(-100), std::runtime_error);
    t1.Lock();
    t1.ChangeBalance(-100);
    EXPECT_EQ(t1.GetBalance(), 400);
    t1.Unlock();
}
```
## 4. Создать директорию .github\workflows и написать actions.yml:

```name: Bank
on: [ push ]
jobs:
  BuildTest:
    runs-on: ubuntu-latest
    timeout-minutes: 3

    steps:
      - name: checkout
        uses: actions/checkout@v3

      - name: prepare
        run: |
          sudo apt-get install lcov
      - name: building
        run: |
          mkdir banking/build && cd banking/build
          cmake -DBUILD_TESTS=ON -DCOVERALLS=ON ..
          make
      - name: test
        run: |
          cd banking/build
          ctest
      - name: coverage
        run: |
          mkdir coverage
          lcov -d . -t bank_test -o coverage/lcov.info  -b . -c --no-external
          lcov --remove coverage/lcov.info '*/gmock/*' '*/gtest/*' '*/include/*' -o coverage/lcov.info
      - name: Coveralls GitHub Action
        uses: coverallsapp/github-action@1.1.3
        with:
          github-token: ${{ github.token }}

```

## 5. Добавить .coveralls.yml
