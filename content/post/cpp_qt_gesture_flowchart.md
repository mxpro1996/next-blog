---
title: "[C/C++] QT手势识别 - 层次笔记"
date: 2025-11-18
draft: false
tags: ["c++", "qt", "手势识别", "框架"]
---

## 挂钩、转发
**职责**: 注册手势类型，启用触摸事件支持

**代码片段**:
```cpp
// 在构造函数中注册需要的手势类型
grabGesture(Qt::PinchGesture);    // 捏合缩放手势
grabGesture(Qt::PanGesture);      // 平移拖动手势  
grabGesture(Qt::SwipeGesture);    // 滑动手势
grabGesture(Qt::TapGesture);      // 点击手势
setAttribute(Qt::WA_AcceptTouchEvents);  // 启用触摸事件

bool MyWidget::event(QEvent *event) 
{
    if (event->type() == QEvent::Gesture) {
        return gestureEvent(static_cast<QGestureEvent*>(event));
    }
    return QWidget::event(event);
}
```

## 分类
**职责**: 按手势类型分发到对应的具体处理器

**代码片段**:
```cpp
bool MyWidget::gestureEvent(QGestureEvent *event) 
{
    if (QGesture *pinchGesture = event->gesture(Qt::PinchGesture)) {
        processPinchGesture(static_cast<QPinchGesture*>(pinchGesture));
    }
    if (QGesture *panGesture = event->gesture(Qt::PanGesture)) {
        processPanGesture(static_cast<QPanGesture*>(panGesture));
    }
    if (QGesture *swipeGesture = event->gesture(Qt::SwipeGesture)) {
        processSwipeGesture(static_cast<QSwipeGesture*>(swipeGesture));
    }
    return true;
}
```

## 分态
**职责**: 管理手势的生命周期状态，处理不同阶段逻辑

**代码片段**:
```cpp
void MyWidget::processPinchGesture(QPinchGesture *gesture) 
{
    switch(gesture->state()) {
    case Qt::GestureStarted:
        // 记录初始状态，准备开始手势
        m_initialScaleFactor = m_currentScaleFactor;
        break;
    case Qt::GestureUpdated:
        // 处理手势进行中的增量变化
        m_tempScaleFactor = m_initialScaleFactor * gesture->totalScaleFactor();
        break;
    case Qt::GestureFinished:
    case Qt::GestureCanceled:
        // 应用最终状态，清理临时数据
        m_currentScaleFactor = m_tempScaleFactor;
        m_tempScaleFactor = 1.0;
        break;
    }
}
```

## 回调
**职责**: 将手势操作映射到具体的业务逻辑

**代码片段**:
```cpp
void MyWidget::onPinchGestureTriggered(qreal scaleFactor) 
{
    // 更新视图缩放
    zoomViewport(scaleFactor);
}

void MyWidget::onPanGestureTriggered(const QPointF &delta) 
{
    // 移动内容位置
    scrollContent(delta);
}

void MyWidget::onSwipeGestureTriggered(QSwipeGesture::SwipeDirection direction) 
{
    // 翻页导航
    switch(direction) {
    case QSwipeGesture::Left:
        navigateToNextPage();
        break;
    case QSwipeGesture::Right:
        navigateToPreviousPage();
        break;
    case QSwipeGesture::Up:
        scrollUp();
        break;
    case QSwipeGesture::Down:
        scrollDown();
        break;
    }
}
```

## 变换计算层
**职责**: 计算几何变换参数，处理边界限制

**代码片段**:
```cpp
void MyWidget::calculateTransform() 
{
    // 缩放计算
    qreal scaleFactor = gesture->totalScaleFactor();
    m_currentScale *= scaleFactor;
    
    // 平移计算
    QPointF delta = gesture->delta();
    m_contentOffset += delta;
    
    // 边界限制
    qreal maxOffsetX = calculateMaxOffsetX();
    qreal maxOffsetY = calculateMaxOffsetY();
    m_contentOffset.setX(qMin(maxOffsetX, qMax(-maxOffsetX, m_contentOffset.x())));
    m_contentOffset.setY(qMin(maxOffsetY, qMax(-maxOffsetY, m_contentOffset.y())));
    
    // 缩放范围限制
    m_currentScale = qMax(m_minScale, qMin(m_maxScale, m_currentScale));
}
```

## 渲染更新层
**职责**: 更新界面显示，应用变换效果 (__每个Widget的实际画板投影，就靠paint回调__)

**代码片段**:
```cpp
void MyWidget::updateDisplay() 
{
    // 触发重绘
    update();
}

void MyWidget::paintEvent(QPaintEvent *event) 
{
    QPainter painter(this);
    
    // 应用变换矩阵
    painter.translate(m_contentOffset);
    painter.scale(m_currentScale, m_currentScale);
    
    // 绘制内容
    drawContent(painter);
    
    // 可选：绘制调试信息
    if (m_showDebugInfo) {
        drawGestureDebugInfo(painter);
    }
}
```
