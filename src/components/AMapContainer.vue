<template>
  <div class="map-container" ref="mapContainer"></div>
  <div v-if="loading" class="loading">加载中...</div>
  <button @click="getLocation" class="location-button">获取位置</button>
  <div v-if="!userPosition && !loading" class="tip-message">请点击右上角"获取位置"按钮开始</div>
  
  <!-- 新增转盘弹窗 -->
  <div v-if="places.length > 0" class="wheel-popup-button">
    <button @click="openWheelPopup" class="open-wheel-button">选择地点</button>
  </div>
  
  <!-- 转盘弹窗 -->
  <div v-if="showWheel" class="wheel-popup-overlay">
    <div class="wheel-popup-content">
      <div class="wheel-popup-header">
        <h3>随机选择地点</h3>
        <button @click="closeWheelPopup" class="close-button">&times;</button>
      </div>
      
      <div class="wheel-popup-body">
        <LuckyWheel
          ref="myLucky"
          width="300px"
          height="300px"
          :prizes="wheelPrizes"
          :blocks="wheelBlocks"
          :buttons="wheelButtons"
          @start="startCallback"
          @end="endCallback"
        />
        
        <div class="selected-place">
          <div v-if="selectedPlace">
            <h3>{{ selectedPlace.name }}</h3>
            <p><i class="location-icon"></i> {{ selectedPlace.address || '无地址信息' }}</p>
            <p><i class="distance-icon"></i> 距离: {{ (selectedPlace.distance / 1000).toFixed(2) }}公里</p>
            <p><i class="type-icon"></i> 类型: {{ selectedPlace.type || '未知' }}</p>
            <p v-if="selectedPlace.pname && selectedPlace.cityname && selectedPlace.adname">
              <i class="area-icon"></i> 
              {{ selectedPlace.pname }} {{ selectedPlace.cityname }} {{ selectedPlace.adname }}
            </p>
          </div>
          <div v-else class="no-selection">
            <p>请点击转盘中心或"开始"按钮选择地点</p>
          </div>
        </div>
      </div>
      
      <div class="wheel-popup-footer">
        <button @click="closeWheelPopup" class="cancel-button">关闭</button>
        <button @click="goToSelectedPlace" v-if="selectedPlace" class="confirm-button">前往地点</button>
      </div>
    </div>
  </div>
</template>

<script setup>
import { ref, onMounted, computed, onUnmounted, nextTick, watch } from 'vue';
import AMapLoader from '@amap/amap-jsapi-loader';
import { LuckyWheel } from '@lucky-canvas/vue';

// 定义变量
const mapContainer = ref(null);
const myLucky = ref(null);
const map = ref(null);
const loading = ref(true);
const places = ref([]);
const userPosition = ref(null);
const selectedPlaceIndex = ref(null);
const isRunning = ref(false);
const showDetail = ref(false); // 控制是否显示详情
const showWheel = ref(false); // 控制显示转盘弹窗
let AMapInstance = null;

// 转盘配置
const wheelBlocks = ref([{ padding: '13px', background: '#617df2' }]);
const wheelButtons = ref([{
  radius: '35%',
  background: '#8a9bf3',
  pointer: true,
  fonts: [{ text: '开始', top: '-10px' }]
}]);

// 转盘奖品数据，从places生成
const wheelPrizes = computed(() => {
  if (!places.value.length) return [];
  
  return places.value.map((place, index) => {
    // 四种交替颜色
    let bgColor;
    if (index % 4 === 0) bgColor = '#e9e8fe';
    else if (index % 4 === 1) bgColor = '#b8c5f2';
    else if (index % 4 === 2) bgColor = '#e9e8fe';
    else bgColor = '#b8c5f2';
    
    return {
      id: index, // 确保每个奖品都有id，与places数组索引对应
      fonts: [{ text: place.name, top: '10%', fontSize: '12px' }],
      background: bgColor
    };
  });
});

// 选中的地点
const selectedPlace = computed(() => {
  if (selectedPlaceIndex.value !== null && places.value.length > 0) {
    return places.value[selectedPlaceIndex.value];
  }
  return null;
});

// POI类型列表
const poiTypes = '050117|050500|080110|080300|080302|080304|080305|080306|080307|080308|080503|110000|110100|110101|110200|110202|110204';

// 地图配置
const mapOptions = {
  zoom: 15,
  center: [116.397428, 39.90923], // 默认中心点，会在获取用户位置后更新
  resizeEnable: true,
  viewMode: '2D'
};

// 打开转盘弹窗
const openWheelPopup = () => {
  showWheel.value = true;
  showDetail.value = false;
};

// 关闭转盘弹窗
const closeWheelPopup = () => {
  showWheel.value = false;
};

// 前往选中的地点
const goToSelectedPlace = () => {
  if (selectedPlace.value && selectedPlace.value.location) {
    const position = [selectedPlace.value.location.split(',')[0], selectedPlace.value.location.split(',')[1]];
    
    // 清除地图上所有标记
    if (map.value) {
      map.value.clearMap();
      
      // 只添加选中地点的标记
      if (AMapInstance) {
        const marker = new AMapInstance.Marker({
          position: position,
          label: {
            content: selectedPlace.value.name,
            direction: 'top'
          },
          icon: new AMapInstance.Icon({
            size: new AMapInstance.Size(25, 34),
            imageSize: new AMapInstance.Size(25, 34),
            image: 'https://webapi.amap.com/theme/v1.3/markers/n/mark_r.png'
          })
        });
        
        map.value.add(marker);
      }
    }
    
    // 设置地图中心并放大
    map.value.setCenter(position);
    map.value.setZoom(17);
    closeWheelPopup();
  }
};

// 转盘开始回调
const startCallback = () => {
  // 防止重复点击
  if (isRunning.value) return;
  
  isRunning.value = true;
  
  // 调用抽奖组件的play方法开始游戏
  myLucky.value.play();
  
  // 随机选择一个索引
  const index = Math.floor(Math.random() * places.value.length);
  console.log('准备选中的索引：', index);
  // 保存我们想要选中的索引，以防无法从回调中获取
  const targetIndex = index;
  
  // 延迟停止
  setTimeout(() => {
    console.log('即将停止转盘，索引：', index);
    // 保存索引以供回调使用
    window._selectedIndex = targetIndex;
    myLucky.value.stop(targetIndex);
  }, 3000);
};

// 转盘结束回调
const endCallback = (prize) => {
  isRunning.value = false;
  console.log('转盘选择结束，选中奖品：', prize);
  
  // 打印关键数据结构
  console.log('完整的prize对象：', JSON.stringify(prize));
  console.log('places数组长度：', places.value.length);
  console.log('wheelPrizes数组：', wheelPrizes.value);
  
  // 尝试所有可能的方式获取索引
  let foundIndex = null;
  
  // 1. 从自定义存储中获取
  if (window._selectedIndex !== undefined) {
    console.log('从window._selectedIndex获取到索引：', window._selectedIndex);
    foundIndex = window._selectedIndex;
  }
  
  // 2. 检查prize是否有index或idx属性
  if (foundIndex === null) {
    try {
      console.log('Prize对象属性:', Object.keys(prize));
      
      if (prize.index !== undefined) {
        console.log('从prize.index获取到索引：', prize.index);
        foundIndex = prize.index;
      } else if (prize.idx !== undefined) {
        console.log('从prize.idx获取到索引：', prize.idx);
        foundIndex = prize.idx;
      } else if (prize.id !== undefined) {
        console.log('从prize.id获取到索引：', prize.id);
        foundIndex = prize.id;
      }
    } catch (e) {
      console.error('获取prize属性出错：', e);
    }
  }
  
  // 3. 检查__luckyOb__属性
  if (foundIndex === null && prize.__luckyOb__) {
    try {
      console.log('Lucky prize __luckyOb__对象属性:', Object.keys(prize.__luckyOb__));
      
      if (prize.__luckyOb__.index !== undefined) {
        console.log('从prize.__luckyOb__.index获取到索引：', prize.__luckyOb__.index);
        foundIndex = prize.__luckyOb__.index;
      } else if (prize.__luckyOb__.id !== undefined) {
        console.log('从prize.__luckyOb__.id获取到索引：', prize.__luckyOb__.id);
        foundIndex = prize.__luckyOb__.id;
      }
    } catch (e) {
      console.error('获取__luckyOb__属性出错：', e);
    }
  }
  
  // 4. 尝试获取转盘组件的当前索引
  if (foundIndex === null && myLucky.value) {
    try {
      console.log('转盘组件：', myLucky.value);
      if (myLucky.value.curIndex !== undefined) {
        console.log('从myLucky.value.curIndex获取到索引：', myLucky.value.curIndex);
        foundIndex = myLucky.value.curIndex;
      }
    } catch (e) {
      console.error('获取转盘组件属性出错：', e);
    }
  }
  
  // 最终检查并设置索引
  if (foundIndex !== null && !isNaN(foundIndex) && foundIndex >= 0 && foundIndex < places.value.length) {
    selectedPlaceIndex.value = foundIndex;
    console.log('成功设置selectedPlaceIndex:', selectedPlaceIndex.value);
    console.log('选中地点详情：', places.value[selectedPlaceIndex.value]);
  } else {
    console.error('无法获取有效的索引，使用默认索引0');
    // 如果实在无法获取有效索引，默认使用第一个地点
    selectedPlaceIndex.value = 0;
    console.log('使用默认索引选中地点详情：', places.value[0]);
  }
};

// 初始化地图
const initMap = async () => {
  try {
    AMapInstance = await AMapLoader.load({
      key: 'e4cc19d2d9747696ef0f9058f0d94112',
      version: '2.0',
      plugins: ['AMap.Geolocation', 'AMap.PlaceSearch', 'AMap.Marker']
    });
    
    // 确保容器已渲染
    await nextTick();
    
    // 创建地图实例
    map.value = new AMapInstance.Map(mapContainer.value, {
      ...mapOptions,
      viewMode: '2D',
      pitch: 0,
      jogEnable: false,
      resizeEnable: true
    });
    
    // 设置地图样式
    map.value.setMapStyle('amap://styles/normal');
    
    // 处理地图加载完成事件
    map.value.on('complete', () => {
      console.log('地图加载完成');
      loading.value = false;
    });
    
    // 添加窗口调整监听，使用passive选项
    window.addEventListener('resize', () => {
      map.value && map.value.resize();
    }, { passive: true });
    
  } catch (e) {
    console.error('地图加载失败', e);
    loading.value = false;
  }
};

// 获取用户位置 - 需要用户手势触发
const getLocation = () => {
  if (!map.value || !AMapInstance) {
    console.error('地图未初始化');
    return;
  }
  
  loading.value = true;
  
  try {
    // 获取用户位置
    const geolocation = new AMapInstance.Geolocation({
      enableHighAccuracy: true,
      timeout: 10000,
      buttonPosition: 'RB',
      buttonOffset: new AMapInstance.Pixel(10, 20),
      zoomToAccuracy: true
    });
    
    map.value.addControl(geolocation);
    
    geolocation.getCurrentPosition((status, result) => {
      if (status === 'complete') {
        userPosition.value = result.position;
        map.value.setCenter(userPosition.value);
        
        // 获取周边POI
        searchNearbyPlaces();
      } else {
        console.error('定位失败', result);
        loading.value = false;
      }
    });
  } catch (e) {
    console.error('获取位置失败', e);
    loading.value = false;
  }
};

// 搜索附近地点
const searchNearbyPlaces = async () => {
  if (!userPosition.value) {
    console.error('用户位置未获取');
    loading.value = false;
    return;
  }
  
  try {
    const allPois = [];
    const totalPages = 2; // 获取2页，每页25条，共50个POI
    const pageSize = 25; // 设置每页数据条数为25（最大值）
    
    for (let pageNum = 1; pageNum <= totalPages; pageNum++) {
      const params = {
        key: 'a845da0ae6bf794ca607bf4439c79f1f',
        location: `${userPosition.value.lng},${userPosition.value.lat}`,
        radius: 5000,
        types: poiTypes,
        sortrule: 'distance',
        page_size: pageSize,
        page_num: pageNum
      };
      
      const queryString = Object.keys(params)
        .map(key => `${key}=${encodeURIComponent(params[key])}`)
        .join('&');
      
      console.log(`请求第${pageNum}页数据，URL: https://restapi.amap.com/v5/place/around?${queryString}`);
      
      const response = await fetch(`https://restapi.amap.com/v5/place/around?${queryString}`);
      const data = await response.json();
      
      console.log(`第${pageNum}页数据返回:`, data);
      
      if (data.status === '1' && data.pois && data.pois.length > 0) {
        allPois.push(...data.pois);
        
        // 如果返回的结果少于页大小，说明已经没有更多数据了
        if (data.pois.length < pageSize) {
          console.log(`第${pageNum}页数据不足${pageSize}条，不再继续请求`);
          break;
        }
      } else {
        console.log(`第${pageNum}页数据获取失败或没有数据`);
        break;
      }
    }
    
    console.log(`总共获取到${allPois.length}个POI点`);
    
    if (allPois.length > 0) {
      places.value = allPois;
      
      // 清除之前的标记点
      if (map.value) {
        map.value.clearMap();
        
        // 在地图上标记这些地点
        places.value.forEach((place, index) => {
          if (place.location) {
            const position = [place.location.split(',')[0], place.location.split(',')[1]];
            
            if (AMapInstance && position[0] && position[1]) {
              const marker = new AMapInstance.Marker({
                position: position,
                label: {
                  content: `${index + 1}. ${place.name}`,
                  direction: 'top'
                }
              });
              
              map.value.add(marker);
            }
          }
        });
      }
    } else {
      console.error('未找到周边地点');
    }
  } catch (e) {
    console.error('搜索附近地点失败', e);
  } finally {
    loading.value = false;
  }
};

// 组件挂载完成
onMounted(() => {
  initMap();
});

// 组件卸载
onUnmounted(() => {
  window.removeEventListener('resize', () => {
    map.value && map.value.resize();
  });
  
  // 销毁地图实例
  if (map.value) {
    map.value.destroy();
    map.value = null;
  }
});
</script>

<style scoped>
.map-container {
  width: 100%;
  height: 100vh;
  position: relative;
  overflow: hidden;
  z-index: 1;
}

.loading {
  position: fixed;
  top: 50%;
  left: 50%;
  transform: translate(-50%, -50%);
  background: rgba(0, 0, 0, 0.7);
  color: white;
  padding: 10px 20px;
  border-radius: 4px;
  z-index: 100;
}

.location-button {
  position: fixed;
  top: 20px;
  right: 20px;
  padding: 10px 20px;
  background: #e74c3c;
  color: white;
  border: none;
  border-radius: 4px;
  cursor: pointer;
  font-size: 16px;
  font-weight: bold;
  z-index: 95;
}

.location-button:hover {
  opacity: 0.9;
}

.tip-message {
  position: fixed;
  top: 50%;
  left: 50%;
  transform: translate(-50%, -50%);
  background: rgba(0, 0, 0, 0.7);
  color: white;
  padding: 15px 25px;
  border-radius: 8px;
  font-size: 16px;
  text-align: center;
  z-index: 90;
  box-shadow: 0 4px 12px rgba(0, 0, 0, 0.2);
}

/* 触发按钮样式 */
.wheel-popup-button {
  position: fixed;
  bottom: 20px;
  right: 20px;
  z-index: 90;
}

.open-wheel-button {
  padding: 12px 20px;
  background: #3498db;
  color: white;
  border: none;
  border-radius: 4px;
  cursor: pointer;
  font-size: 16px;
  font-weight: bold;
  box-shadow: 0 2px 10px rgba(0, 0, 0, 0.2);
}

.open-wheel-button:hover {
  opacity: 0.9;
}

/* 转盘弹窗样式 */
.wheel-popup-overlay {
  position: fixed;
  top: 0;
  left: 0;
  width: 100%;
  height: 100%;
  background: rgba(0, 0, 0, 0.7);
  display: flex;
  justify-content: center;
  align-items: center;
  z-index: 999;
}

.wheel-popup-content {
  background: white;
  border-radius: 8px;
  width: 90%;
  max-width: 500px;
  max-height: 90vh;
  overflow-y: auto;
  display: flex;
  flex-direction: column;
  box-shadow: 0 5px 20px rgba(0, 0, 0, 0.3);
  animation: popup 0.3s ease-out;
}

@keyframes popup {
  from { transform: scale(0.9); opacity: 0; }
  to { transform: scale(1); opacity: 1; }
}

.wheel-popup-header {
  padding: 15px;
  border-bottom: 1px solid #eee;
  display: flex;
  justify-content: space-between;
  align-items: center;
}

.wheel-popup-header h3 {
  margin: 0;
  color: #333;
}

.close-button {
  background: none;
  border: none;
  font-size: 24px;
  cursor: pointer;
  color: #999;
}

.wheel-popup-body {
  padding: 20px;
  display: flex;
  flex-direction: column;
  align-items: center;
}

.wheel-popup-footer {
  padding: 15px;
  border-top: 1px solid #eee;
  display: flex;
  justify-content: flex-end;
  gap: 10px;
}

.cancel-button, .confirm-button {
  padding: 8px 15px;
  border: none;
  border-radius: 4px;
  cursor: pointer;
  font-size: 14px;
}

.cancel-button {
  background: #e0e0e0;
  color: #333;
}

.confirm-button {
  background: #3498db;
  color: white;
}

.selected-place {
  width: 100%;
  background: rgba(255, 255, 255, 0.95);
  padding: 15px;
  border-radius: 8px;
  box-shadow: 0 4px 15px rgba(0, 0, 0, 0.15);
  margin-top: 20px;
  animation: fadeIn 0.5s ease-in-out;
  min-height: 120px; /* 确保有足够的高度 */
}

@keyframes fadeIn {
  from { opacity: 0; transform: translateY(10px); }
  to { opacity: 1; transform: translateY(0); }
}

.selected-place h3 {
  margin: 0 0 10px 0;
  font-size: 18px;
  color: #333;
  text-align: center;
}

.selected-place p {
  margin: 6px 0;
  font-size: 14px;
  color: #666;
  display: flex;
  align-items: center;
}

.location-icon, .distance-icon, .type-icon, .area-icon {
  display: inline-block;
  width: 16px;
  height: 16px;
  margin-right: 8px;
  background-color: #3498db;
  border-radius: 50%;
}

.distance-icon {
  background-color: #e74c3c;
}

.type-icon {
  background-color: #2ecc71;
}

.area-icon {
  background-color: #f39c12;
}

.no-selection {
  display: flex;
  justify-content: center;
  align-items: center;
  color: #999;
  font-style: italic;
  height: 100px;
}
</style> 