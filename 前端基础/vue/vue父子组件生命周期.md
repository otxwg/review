### 加载渲染过程
父beforecreate -》父create -》父beforemouted -》子beforecreate -》子create-》子beforemouted-》子mouted-》父mouted 

### 子更新过程
父 beforeUpdate-》子 beforeUpdate -》子update-》父update

### 父更新过程
父 beforeUpdate-》父update

### 销毁过程

父 beforedestroy-》子 beforedestroy -》子destroy-》父destroy