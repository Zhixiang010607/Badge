<script setup lang="ts">
import type {FormInstance, FormRules} from 'element-plus'
import {Cropper, CropperResult} from 'vue-advanced-cropper'
import 'vue-advanced-cropper/dist/style.css'
import {CopyDocument, Crop, Delete, Edit, List, Plus} from '@element-plus/icons-vue'
import {computed} from 'vue'

type ShapeType = 'circle' | 'rectangle' | 'ellipse' | 'polygon' | 'redHeart' | 'yellowHeart'

interface RuleForm {
  shape: ShapeType
  row: number
  col: number
  diam: number
  rectWidth: number
  rectHeight: number
  ellipseLong: number
  ellipseShort: number
  polygonSides: number
  polygonSide: number
  padding: number
  color: string
  width: number
  height: number
}

interface PaperSize {
  width: number
  height: number
}

interface ExportForm {
  fileName: string
  fileType: string
  quality: number
  dpi: number
}

interface ShapeSize {
  width: number
  height: number
}

interface ShapeBounds extends ShapeSize {
  minX: number
  minY: number
}

interface ShapeLayout {
  inner: ShapeSize
  outer: ShapeSize
  offsetX: number
  offsetY: number
}

interface AuthUser {
  id?: string
  username: string
  role: 'admin' | 'employee'
  token: string
}

interface EmployeeAccount {
  id: string
  username: string
  password?: string
  createdAt: string
}

const AUTH_SESSION_KEY = 'badge-print-session-v1'
const AUTH_REMEMBER_KEY = 'badge-print-remember-login-v1'
const ADMIN_USERNAME = 'yanyujie123'
const windowWithAuthConfig = window as Window & {
  BADGE_AUTH_API_URL?: string
  YYJ_AUTH_API_URL?: string
}
const AUTH_API_BASE_URL = (
  windowWithAuthConfig.BADGE_AUTH_API_URL ||
  windowWithAuthConfig.YYJ_AUTH_API_URL ||
  document.querySelector<HTMLMetaElement>('meta[name="auth-api-base"]')?.content ||
  ''
).replace(/\/$/, '')

const paperSize: Map<string, PaperSize> = new Map([
  ['A3', {
    width: 7016,
    height: 9921
  }],
  ['A4', {
    width: 4961,
    height: 7016
  }],
  ['A5', {
    width: 3496,
    height: 4961
  }],
  ['自定义', {
    width: 4961,
    height: 7016
  }],
])

const shapeOptions: Array<{ label: string, value: ShapeType }> = [
  {label: '圆形', value: 'circle'},
  {label: '矩形', value: 'rectangle'},
  {label: '椭圆', value: 'ellipse'},
  {label: '正多边形', value: 'polygon'},
  {label: '红色心形', value: 'redHeart'},
  {label: '黄色心形', value: 'yellowHeart'},
]

const fixedShapeSize: Record<'redHeart' | 'yellowHeart', ShapeSize> = {
  redHeart: {
    width: 6,
    height: 5.46
  },
  yellowHeart: {
    width: 2.5,
    height: 2.13
  }
}

const paper = ref<string>('A4')

watch(paper, (val) => {
  ruleForm.width = paperSize.get(val)?.width || 0
  ruleForm.height = paperSize.get(val)?.height || 0
})

const ruleFormRef = ref<FormInstance>()
const ruleForm = reactive<RuleForm>({
  shape: 'circle',
  diam: 6.6,
  rectWidth: 5,
  rectHeight: 7,
  ellipseLong: 7,
  ellipseShort: 5,
  polygonSides: 5,
  polygonSide: 4,
  padding: 0.2,
  width: 4961,
  height: 7016,
  row: 4,
  col: 2,
  color: '#DDDDDD'
})

const exportForm = reactive<ExportForm>({
  fileName: '吧唧图',
  fileType: 'png',
  quality: 1,
  dpi: 600
})

const rules = reactive<FormRules<RuleForm>>({
  diam: [
    {required: true, message: '请输入直径', trigger: 'blur'}
  ],
  rectWidth: [
    {required: true, message: '请输入矩形宽度', trigger: 'blur'}
  ],
  rectHeight: [
    {required: true, message: '请输入矩形高度', trigger: 'blur'}
  ],
  ellipseLong: [
    {required: true, message: '请输入椭圆长边', trigger: 'blur'}
  ],
  ellipseShort: [
    {required: true, message: '请输入椭圆短边', trigger: 'blur'}
  ],
  polygonSides: [
    {required: true, message: '请输入边数', trigger: 'blur'}
  ],
  polygonSide: [
    {required: true, message: '请输入边长', trigger: 'blur'}
  ],
  padding: [
    {required: true, message: '请输入边距', trigger: 'blur'}
  ],
  row: [
    {required: true, message: '请输入行数', trigger: 'blur'}
  ],
  col: [
    {required: true, message: '请输入列数', trigger: 'blur'}
  ],
  color: [
    {required: true, message: '请选择边框颜色', trigger: 'blur'}
  ],
  width: [
    {required: true, message: '请输入纸张宽度', trigger: 'blur'}
  ],
  height: [
    {required: true, message: '请输入纸张高度', trigger: 'blur'}
  ],
})

const tempImages = ref<Array<Array<string>>>([])
const images = ref<Array<Array<string>>>([])
const previewBox = ref<HTMLElement>()
const preview = ref<HTMLElement>()
const badgeBg = ref<HTMLElement[]>([])
const badge = ref<HTMLElement[]>([])

const btnScale = ref(1)
const previewScale = ref(1)

const submitForm = ref<RuleForm>({...ruleForm})
const currentUser = ref<AuthUser | null>(null)
const authReady = ref(false)
const loginLoading = ref(false)
const loginForm = reactive({
  username: '',
  password: '',
  remember: false
})
const loginMessage = ref('')
const adminVisible = ref(false)
const employeeMessage = ref('')
const employeeForm = reactive({
  username: '',
  password: ''
})
const employeeAccounts = ref<EmployeeAccount[]>([])

const readRememberedLogin = () => {
  try {
    const remembered = JSON.parse(window.localStorage.getItem(AUTH_REMEMBER_KEY) || 'null')
    if (!remembered || typeof remembered.username !== 'string' || typeof remembered.password !== 'string') {
      return
    }
    loginForm.username = remembered.username
    loginForm.password = remembered.password
    loginForm.remember = true
  } catch {
    window.localStorage.removeItem(AUTH_REMEMBER_KEY)
  }
}

const saveRememberedLogin = () => {
  if (!loginForm.remember) {
    window.localStorage.removeItem(AUTH_REMEMBER_KEY)
    return
  }
  window.localStorage.setItem(AUTH_REMEMBER_KEY, JSON.stringify({
    username: loginForm.username.trim(),
    password: loginForm.password
  }))
}

const authApiUrl = (path: string) => {
  return `${AUTH_API_BASE_URL}${path}`
}

const readStoredToken = () => {
  try {
    const session = JSON.parse(window.sessionStorage.getItem(AUTH_SESSION_KEY) || 'null')
    return typeof session?.token === 'string' ? session.token : ''
  } catch {
    return ''
  }
}

const authApiRequest = async (
  path: string,
  options: {
    auth?: boolean
    token?: string
    method?: string
    headers?: Record<string, string>
    body?: unknown
  } = {}
) => {
  const headers: Record<string, string> = {
    Accept: 'application/json',
    ...(options.headers || {})
  }
  const token = options.token || (options.auth === false ? '' : currentUser.value?.token || readStoredToken())
  if (token) {
    headers.Authorization = `Bearer ${token}`
  }
  const requestOptions: RequestInit = {
    method: options.method || 'GET',
    headers
  }
  if (options.body !== undefined) {
    headers['Content-Type'] = 'application/json'
    requestOptions.body = JSON.stringify(options.body)
  }

  let response: Response
  try {
    response = await fetch(authApiUrl(path), requestOptions)
  } catch {
    throw new Error('云端账号 API 连接失败，请检查 Worker 地址是否配置正确')
  }

  const text = await response.text()
  let data: any = {}
  try {
    data = text ? JSON.parse(text) : {}
  } catch {
    throw new Error('云端账号 API 返回异常，请检查 Worker 是否部署成功')
  }
  if (!response.ok) {
    throw new Error(data.error || '云端账号请求失败')
  }
  return data
}

const readSessionUser = async (): Promise<AuthUser | null> => {
  try {
    const token = readStoredToken()
    if (!token) {
      return null
    }
    const data = await authApiRequest('/api/auth/me', {
      auth: false,
      token
    })
    return {
      ...data.user,
      token
    }
  } catch {
    window.sessionStorage.removeItem(AUTH_SESSION_KEY)
    return null
  }
}

const writeSessionUser = (user: AuthUser) => {
  window.sessionStorage.setItem(AUTH_SESSION_KEY, JSON.stringify({
    token: user.token
  }))
}

const authenticate = async (username: string, password: string): Promise<AuthUser> => {
  const data = await authApiRequest('/api/auth/login', {
    auth: false,
    method: 'POST',
    body: {
      username,
      password
    }
  })
  return {
    ...data.user,
    token: data.token
  }
}

const login = async () => {
  loginLoading.value = true
  loginMessage.value = '正在登录...'
  try {
    const user = await authenticate(loginForm.username.trim(), loginForm.password)
    currentUser.value = user
    writeSessionUser(user)
    saveRememberedLogin()
    loginMessage.value = ''
    if (!loginForm.remember) {
      loginForm.password = ''
    }
    nextTick(calcPaper)
  } catch (error: any) {
    loginMessage.value = error?.message || '账号或密码不正确'
  } finally {
    loginLoading.value = false
  }
}

const logout = () => {
  window.sessionStorage.removeItem(AUTH_SESSION_KEY)
  currentUser.value = null
  adminVisible.value = false
  loginMessage.value = '已退出登录'
}

const refreshEmployeeAccounts = async () => {
  if (currentUser.value?.role !== 'admin') {
    employeeAccounts.value = []
    return
  }
  setEmployeeMessage('正在读取员工账号...')
  try {
    const data = await authApiRequest('/api/employees')
    employeeAccounts.value = data.employees || []
    setEmployeeMessage('')
  } catch (error: any) {
    employeeAccounts.value = []
    setEmployeeMessage(error?.message || '无法读取员工账号')
  }
}

const toggleAdminPanel = () => {
  adminVisible.value = !adminVisible.value
  if (adminVisible.value) {
    refreshEmployeeAccounts()
  }
}

const setEmployeeMessage = (message: string) => {
  employeeMessage.value = message
}

const createSuggestedUsername = () => {
  return `staff${Math.floor(1000 + Math.random() * 9000)}`
}

const createSuggestedPassword = () => {
  return Math.random().toString(36).slice(2, 10)
}

const generateEmployeeCredentials = () => {
  employeeForm.username = createSuggestedUsername()
  employeeForm.password = createSuggestedPassword()
  setEmployeeMessage('已随机生成账号和密码')
}

const createEmployeeAccount = async () => {
  const username = employeeForm.username.trim()
  const password = employeeForm.password.trim()
  if (!username || !password) {
    setEmployeeMessage('请输入员工账号和密码')
    return
  }
  if (username === ADMIN_USERNAME) {
    setEmployeeMessage('员工账号不能和管理员账号相同')
    return
  }
  setEmployeeMessage('正在新增员工账号...')
  try {
    await authApiRequest('/api/employees', {
      method: 'POST',
      body: {
        username,
        password
      }
    })
    employeeForm.username = ''
    employeeForm.password = ''
    await refreshEmployeeAccounts()
    setEmployeeMessage('员工账号已新增')
  } catch (error: any) {
    setEmployeeMessage(error?.message || '新增员工账号失败')
  }
}

const updateEmployeeAccount = async (account: EmployeeAccount, password = '') => {
  const username = account.username.trim()
  if (!username) {
    setEmployeeMessage('员工账号不能为空')
    return
  }
  if (username === ADMIN_USERNAME) {
    setEmployeeMessage('员工账号不能和管理员账号相同')
    return
  }
  setEmployeeMessage('正在保存员工账号...')
  try {
    await authApiRequest(`/api/employees/${encodeURIComponent(account.id)}`, {
      method: 'PUT',
      body: password.trim()
        ? {
            username,
            password: password.trim()
          }
        : {
            username
          }
    })
    await refreshEmployeeAccounts()
    setEmployeeMessage('员工账号已更新')
  } catch (error: any) {
    setEmployeeMessage(error?.message || '员工账号更新失败')
  }
}

const updateEmployeeDraft = (id: string, key: 'username' | 'password', value: string) => {
  employeeAccounts.value = employeeAccounts.value.map((account) => {
    if (account.id !== id) {
      return account
    }
    return {
      ...account,
      [key]: value
    }
  })
}

const removeEmployeeAccount = async (id: string) => {
  setEmployeeMessage('正在删除员工账号...')
  try {
    await authApiRequest(`/api/employees/${encodeURIComponent(id)}`, {
      method: 'DELETE'
    })
    await refreshEmployeeAccounts()
    setEmployeeMessage('员工账号已删除')
  } catch (error: any) {
    setEmployeeMessage(error?.message || '员工账号删除失败')
  }
}

const cmTo600Dpi = (cm: number) => {
  return cm / 2.54 * 600
}

const getPolygonSides = (sides: number) => {
  return Math.max(3, Math.round(sides))
}

const getRegularPolygonPoints = (sides: number) => {
  const count = getPolygonSides(sides)
  const startAngle = count === 4 ? -3 * Math.PI / 4 : count === 6 ? 0 : -Math.PI / 2
  const points = []
  for (let i = 0; i < count; i++) {
    const angle = startAngle + (2 * Math.PI * i) / count
    points.push({
      x: Math.cos(angle),
      y: Math.sin(angle)
    })
  }
  return points
}

const getBoundsFromPoints = (points: Array<{ x: number, y: number }>, sideLength: number): ShapeBounds => {
  const xs = points.map(point => point.x)
  const ys = points.map(point => point.y)
  const unitWidth = Math.max(...xs) - Math.min(...xs)
  const unitHeight = Math.max(...ys) - Math.min(...ys)
  const unitMinX = Math.min(...xs)
  const unitMinY = Math.min(...ys)
  const unitSide = Math.hypot(points[1].x - points[0].x, points[1].y - points[0].y)
  const scale = sideLength / unitSide
  return {
    minX: unitMinX * scale,
    minY: unitMinY * scale,
    width: unitWidth * scale,
    height: unitHeight * scale
  }
}

const getRegularPolygonBounds = (sides: number, sideLength: number): ShapeBounds => {
  return getBoundsFromPoints(getRegularPolygonPoints(sides), sideLength)
}

const getRegularPolygonMetrics = (sides: number, sideLength: number): ShapeSize => {
  const {width, height} = getRegularPolygonBounds(sides, sideLength)
  return {
    width,
    height
  }
}

const getRegularPolygonClipPath = (sides: number) => {
  const points = getRegularPolygonPoints(sides)
  return getClipPathFromPoints(points)
}

const getClipPathFromPoints = (points: Array<{ x: number, y: number }>) => {
  const xs = points.map(point => point.x)
  const ys = points.map(point => point.y)
  const minX = Math.min(...xs)
  const minY = Math.min(...ys)
  const width = Math.max(...xs) - minX
  const height = Math.max(...ys) - minY
  const path = points.map(point => {
    const x = ((point.x - minX) / width * 100).toFixed(2)
    const y = ((point.y - minY) / height * 100).toFixed(2)
    return `${x}% ${y}%`
  }).join(', ')
  return `polygon(${path})`
}

const getShapeSize = (form: RuleForm): ShapeSize => {
  switch (form.shape) {
    case 'rectangle':
      return {
        width: form.rectWidth,
        height: form.rectHeight
      }
    case 'ellipse':
      return {
        width: Math.max(form.ellipseLong, form.ellipseShort),
        height: Math.min(form.ellipseLong, form.ellipseShort)
      }
    case 'polygon':
      return getRegularPolygonMetrics(form.polygonSides, form.polygonSide)
    case 'redHeart':
    case 'yellowHeart':
      return fixedShapeSize[form.shape]
    case 'circle':
    default:
      return {
        width: form.diam,
        height: form.diam
      }
  }
}

const getShapeOuterSize = (form: RuleForm): ShapeSize => {
  return getShapeLayout(form).outer
}

const getShapeLayout = (form: RuleForm): ShapeLayout => {
  const inner = getShapeSize(form)
  if (form.shape === 'polygon') {
    const sides = getPolygonSides(form.polygonSides)
    const outerSide = form.polygonSide + 2 * form.padding * Math.tan(Math.PI / sides)
    const innerBounds = getRegularPolygonBounds(sides, form.polygonSide)
    const outerBounds = getRegularPolygonBounds(sides, outerSide)
    return {
      inner,
      outer: {
        width: outerBounds.width,
        height: outerBounds.height
      },
      offsetX: innerBounds.minX - outerBounds.minX,
      offsetY: innerBounds.minY - outerBounds.minY
    }
  }
  return {
    inner,
    outer: {
      width: inner.width + form.padding * 2,
      height: inner.height + form.padding * 2
    },
    offsetX: form.padding,
    offsetY: form.padding
  }
}

const getShapeClass = (shape: ShapeType) => {
  return `shape-${shape}`
}

const getShapeStyle = (form: RuleForm) => {
  if (form.shape === 'polygon') {
    return {
      clipPath: getRegularPolygonClipPath(form.polygonSides)
    }
  }
  if (form.shape === 'redHeart' || form.shape === 'yellowHeart') {
    const maskUrl = form.shape === 'redHeart' ? '/red-heart-mask.png' : '/yellow-heart-mask.png'
    return {
      maskImage: `url("${maskUrl}")`,
      maskRepeat: 'no-repeat',
      maskSize: '100% 100%',
      maskPosition: 'center',
      WebkitMaskImage: `url("${maskUrl}")`,
      WebkitMaskRepeat: 'no-repeat',
      WebkitMaskSize: '100% 100%',
      WebkitMaskPosition: 'center'
    }
  }
  return {}
}

const onSubmit = () => {
  if (images.value.every(i => i.every(j => !j))) {
    validateForm()
    return
  }
  ElMessageBox.confirm(
    '已选图片将会被清空，是否继续？',
    '提示',
    {
      confirmButtonText: '确定',
      cancelButtonText: '取消',
      type: 'warning',
    }
  ).then(() => {
    validateForm()
  })
}

const validateForm = () => {
  ruleFormRef.value?.validate((valid) => {
    if (valid) {
      const {row, col, width, height} = ruleForm
      const shapeSize = getShapeOuterSize(ruleForm)
      const actWidth = cmTo600Dpi(col * shapeSize.width)
      if (actWidth > width) {
        ElMessage.warning('列数过多，无法排版')
        return
      }
      const actHeight = cmTo600Dpi(row * shapeSize.height)
      if (actHeight > height) {
        ElMessage.warning('行数过多，无法排版')
        return
      }

      const arr = []
      for (let i = 0; i < row; i++) {
        const rowArr = []
        for (let j = 0; j < col; j++) {
          rowArr.push('')
        }
        arr.push(rowArr)
      }
      images.value = arr
      tempImages.value = JSON.parse(JSON.stringify(arr))

      submitForm.value = {...ruleForm}
      calcPaper()
    }
  })
}

const calcPaper = () => {
  const {color, width, height} = submitForm.value
  const shapeLayout = getShapeLayout(submitForm.value)

  const previewBoxWidth = previewBox.value ? previewBox.value.offsetWidth - 190 : 0
  const previewBoxHeight = previewBox.value ? previewBox.value.offsetHeight - 190 : 0

  let scale = 1
  if (previewBoxWidth / previewBoxHeight > width / height) {
    scale = previewBoxHeight / height
  } else {
    scale = previewBoxWidth / width
  }
  nextTick(() => {
    if (preview.value) {
      preview.value.style.height = `${height}px`
      preview.value.style.width = `${width}px`
      preview.value.style.minHeight = `${height}px`
      preview.value.style.minWidth = `${width}px`
      preview.value.style.maxHeight = `${height}px`
      preview.value.style.maxWidth = `${width}px`
      preview.value.style.transform = `scale(${scale})`
    }
    previewScale.value = scale
    badgeBg.value.forEach(item => {
      item.style.width = `${cmTo600Dpi(shapeLayout.outer.width)}px`
      item.style.height = `${cmTo600Dpi(shapeLayout.outer.height)}px`
      item.style.background = 'transparent'
      item.style.setProperty('--badge-fill-color', color)
      if (submitForm.value.shape === 'polygon') {
        item.style.display = 'block'
        item.style.position = 'relative'
      } else {
        item.style.display = ''
        item.style.position = ''
      }
    })
    badge.value.forEach(item => {
      item.style.width = `${cmTo600Dpi(shapeLayout.inner.width)}px`
      item.style.height = `${cmTo600Dpi(shapeLayout.inner.height)}px`
      if (submitForm.value.shape === 'polygon') {
        item.style.position = 'absolute'
        item.style.left = `${cmTo600Dpi(shapeLayout.offsetX)}px`
        item.style.top = `${cmTo600Dpi(shapeLayout.offsetY)}px`
      } else {
        item.style.position = ''
        item.style.left = ''
        item.style.top = ''
      }
    })
    btnScale.value = 1 / scale
  })
}

onMounted(async () => {
  readRememberedLogin()
  currentUser.value = await readSessionUser()
  authReady.value = true
  calcPaper()
  window.addEventListener('resize', calcPaper)
})
onUnmounted(() => {
  window.removeEventListener('resize', calcPaper)
})

const fileInput = ref<HTMLInputElement>()
const dialogVisible = ref<boolean>(false)
const rowIndex = ref(0)
const colIndex = ref(0)
const editable = ref('')
const actionHover = ref('')
const addImage = (i: number, j: number) => {
  editable.value = ''
  rowIndex.value = i
  colIndex.value = j
  const inputDom = fileInput.value
  if (inputDom) {
    inputDom.value = ''
    inputDom.click()
  }
}
const fileChange = (e: any) => {
  const file = e.target.files[0]
  if (!file) {
    return
  }
  const reader = new FileReader()
  reader.readAsDataURL(file)
  reader.onload = () => {
    images.value[rowIndex.value][colIndex.value] = reader.result as string
    tempImages.value[rowIndex.value][colIndex.value] = reader.result as string
    dialogVisible.value = true
  }
}

const editImage = (i: number, j: number) => {
  editable.value = ''
  rowIndex.value = i
  colIndex.value = j
  dialogVisible.value = true
}
const cropperRef = ref()
const confirmCrop = () => {
  if (!cropperRef.value) {
    return
  }
  const {canvas} = cropperRef.value.getResult() as CropperResult
  images.value[rowIndex.value][colIndex.value] = canvas?.toDataURL() || ''
  dialogVisible.value = false
}

const drawCoverImage = (ctx: CanvasRenderingContext2D, image: HTMLImageElement, x: number, y: number, width: number, height: number) => {
  const sourceRatio = image.naturalWidth / image.naturalHeight
  const targetRatio = width / height
  let sx = 0
  let sy = 0
  let sw = image.naturalWidth
  let sh = image.naturalHeight
  if (sourceRatio > targetRatio) {
    sw = image.naturalHeight * targetRatio
    sx = (image.naturalWidth - sw) / 2
  } else {
    sh = image.naturalWidth / targetRatio
    sy = (image.naturalHeight - sh) / 2
  }
  ctx.drawImage(image, sx, sy, sw, sh, x, y, width, height)
}

const getNormalizedPolygonPoints = (sides: number) => {
  const points = getRegularPolygonPoints(sides)
  const xs = points.map(point => point.x)
  const ys = points.map(point => point.y)
  const minX = Math.min(...xs)
  const minY = Math.min(...ys)
  const width = Math.max(...xs) - minX
  const height = Math.max(...ys) - minY
  return points.map(point => ({
    x: (point.x - minX) / width,
    y: (point.y - minY) / height
  }))
}

const makeShapePath = (ctx: CanvasRenderingContext2D, form: RuleForm, x: number, y: number, width: number, height: number) => {
  ctx.beginPath()
  if (form.shape === 'circle' || form.shape === 'ellipse') {
    ctx.ellipse(x + width / 2, y + height / 2, width / 2, height / 2, 0, 0, Math.PI * 2)
    return
  }
  if (form.shape === 'polygon') {
    const points = getNormalizedPolygonPoints(form.polygonSides)
    points.forEach((point, index) => {
      const px = x + point.x * width
      const py = y + point.y * height
      if (index === 0) {
        ctx.moveTo(px, py)
      } else {
        ctx.lineTo(px, py)
      }
    })
    ctx.closePath()
    return
  }
  ctx.rect(x, y, width, height)
}

const drawMaskedRect = async (
  ctx: CanvasRenderingContext2D,
  maskSrc: string,
  drawContent: (maskCtx: CanvasRenderingContext2D) => void,
  x: number,
  y: number,
  width: number,
  height: number
) => {
  const mask = await loadImg(maskSrc)
  const temp = document.createElement('canvas')
  temp.width = Math.max(1, Math.round(width))
  temp.height = Math.max(1, Math.round(height))
  const tempCtx = temp.getContext('2d')
  if (!tempCtx) {
    return
  }
  drawContent(tempCtx)
  tempCtx.globalCompositeOperation = 'destination-in'
  tempCtx.drawImage(mask, 0, 0, temp.width, temp.height)
  tempCtx.globalCompositeOperation = 'source-over'
  ctx.drawImage(temp, x, y, width, height)
}

const getHeartMaskSrc = (shape: ShapeType) => {
  return shape === 'redHeart' ? '/red-heart-mask.png' : '/yellow-heart-mask.png'
}

const drawShapeFill = async (
  ctx: CanvasRenderingContext2D,
  form: RuleForm,
  color: string,
  x: number,
  y: number,
  width: number,
  height: number
) => {
  if (form.shape === 'redHeart' || form.shape === 'yellowHeart') {
    await drawMaskedRect(ctx, getHeartMaskSrc(form.shape), (maskCtx) => {
      maskCtx.fillStyle = color
      maskCtx.fillRect(0, 0, width, height)
    }, x, y, width, height)
    return
  }
  ctx.save()
  makeShapePath(ctx, form, x, y, width, height)
  ctx.fillStyle = color
  ctx.fill()
  ctx.restore()
}

const drawShapeImage = async (
  ctx: CanvasRenderingContext2D,
  form: RuleForm,
  src: string,
  x: number,
  y: number,
  width: number,
  height: number
) => {
  const image = await loadImg(src)
  if (form.shape === 'redHeart' || form.shape === 'yellowHeart') {
    await drawMaskedRect(ctx, getHeartMaskSrc(form.shape), (maskCtx) => {
      drawCoverImage(maskCtx, image, 0, 0, width, height)
    }, x, y, width, height)
    return
  }
  ctx.save()
  makeShapePath(ctx, form, x, y, width, height)
  ctx.clip()
  drawCoverImage(ctx, image, x, y, width, height)
  ctx.restore()
}

const renderExportCanvas = async () => {
  const {row, col, color, width, height} = submitForm.value
  const dpiScale = exportForm.dpi / 600
  const shapeLayout = getShapeLayout(submitForm.value)
  const outerWidth = cmTo600Dpi(shapeLayout.outer.width)
  const outerHeight = cmTo600Dpi(shapeLayout.outer.height)
  const innerWidth = cmTo600Dpi(shapeLayout.inner.width)
  const innerHeight = cmTo600Dpi(shapeLayout.inner.height)
  const offsetX = cmTo600Dpi(shapeLayout.offsetX)
  const offsetY = cmTo600Dpi(shapeLayout.offsetY)
  const gapX = (width - col * outerWidth) / (col + 1)
  const gapY = (height - row * outerHeight) / (row + 1)
  const canvas = document.createElement('canvas')
  canvas.width = Math.round(width * dpiScale)
  canvas.height = Math.round(height * dpiScale)
  const ctx = canvas.getContext('2d')
  if (!ctx) {
    throw new Error('无法创建导出画布')
  }
  ctx.scale(dpiScale, dpiScale)
  ctx.fillStyle = '#fff'
  ctx.fillRect(0, 0, width, height)
  for (let i = 0; i < row; i++) {
    for (let j = 0; j < col; j++) {
      const image = images.value[i]?.[j]
      if (!image) {
        continue
      }
      const outerX = gapX * (j + 1) + outerWidth * j
      const outerY = gapY * (i + 1) + outerHeight * i
      await drawShapeFill(ctx, submitForm.value, color, outerX, outerY, outerWidth, outerHeight)
      await drawShapeImage(ctx, submitForm.value, image, outerX + offsetX, outerY + offsetY, innerWidth, innerHeight)
    }
  }
  return canvas
}

const loading = ref(false)
const output = () => {
  if (loading.value) {
    return
  }
  loading.value = true
  nextTick(async () => {
    try {
      const canvas = await renderExportCanvas()
      const img = await loadImg(canvas.toDataURL(`image/${exportForm.fileType}`, exportForm.quality))
      const a = document.createElement('a')
      a.href = img.src
      a.download = `${exportForm.fileName || '吧唧图'}.${exportForm.fileType}`
      a.click()
      a.remove()
      img.remove()
      loading.value = false
    } catch (e) {
      loading.value = false
      console.warn(e)
    }
  })
}
const copyItem = ref()
const copyTemp = ref()
const copyImage = (i: number, j: number) => {
  copyItem.value = JSON.parse(JSON.stringify(images.value[i][j]))
  copyTemp.value = JSON.parse(JSON.stringify(tempImages.value[i][j]))
}

const pasteImage = (i: number, j: number) => {
  images.value[i][j] = JSON.parse(JSON.stringify(copyItem.value))
  tempImages.value[i][j] = JSON.parse(JSON.stringify(copyTemp.value))
}

const removeImage = (i: number, j: number) => {
  images.value[i][j] = ''
  tempImages.value[i][j] = ''
}

const cloneValue = (value: string) => JSON.parse(JSON.stringify(value || ''))

const hasCopiedImage = () => {
  return Boolean(copyItem.value)
}

const fillRowFromCopiedImage = (rowIndex: number) => {
  if (!hasCopiedImage()) {
    ElMessage.warning('请先复制一张图片')
    return
  }
  const image = cloneValue(copyItem.value)
  const temp = cloneValue(copyTemp.value)
  images.value[rowIndex] = images.value[rowIndex].map(() => cloneValue(image))
  tempImages.value[rowIndex] = tempImages.value[rowIndex].map(() => cloneValue(temp))
  ElMessage.success('已粘贴到当前行')
}

const fillColumnFromCopiedImage = (colIndex: number) => {
  if (!hasCopiedImage()) {
    ElMessage.warning('请先复制一张图片')
    return
  }
  const image = cloneValue(copyItem.value)
  const temp = cloneValue(copyTemp.value)
  images.value.forEach((row, rowIndex) => {
    row[colIndex] = cloneValue(image)
    tempImages.value[rowIndex][colIndex] = cloneValue(temp)
  })
  ElMessage.success('已粘贴到当前列')
}

const fillAllFromCopiedImage = () => {
  if (!hasCopiedImage()) {
    ElMessage.warning('请先复制一张图片')
    return
  }
  const image = cloneValue(copyItem.value)
  const temp = cloneValue(copyTemp.value)
  images.value = images.value.map((row) => row.map(() => cloneValue(image)))
  tempImages.value = tempImages.value.map((row) => row.map(() => cloneValue(temp)))
  ElMessage.success('已粘贴到全部位置')
}

const clearRow = (rowIndex: number) => {
  images.value[rowIndex] = images.value[rowIndex].map(() => '')
  tempImages.value[rowIndex] = tempImages.value[rowIndex].map(() => '')
  ElMessage.success('已清空当前行')
}

const clearColumn = (colIndex: number) => {
  images.value.forEach((row, rowIndex) => {
    row[colIndex] = ''
    tempImages.value[rowIndex][colIndex] = ''
  })
  ElMessage.success('已清空当前列')
}

const clearAllImages = () => {
  images.value = images.value.map((row) => row.map(() => ''))
  tempImages.value = tempImages.value.map((row) => row.map(() => ''))
  copyItem.value = ''
  copyTemp.value = ''
  ElMessage.success('已全部清空')
}

const loadImg = (src: string): Promise<HTMLImageElement> => {
  return new Promise((resolve, reject) => {
    const img = new Image()
    img.src = src
    img.onload = () => resolve(img)
    img.onerror = reject
  })
}

const goGitHub = () => {
  window.open('https://github.com/HaoHaoP/badge-print')
}

const badgeActionStyle = computed(() => {
  return {
    '--badge-action-scale': btnScale.value
  }
})

const isEditableBadge = (i: number, j: number) => {
  return editable.value === `${i},${j}`
}

const isActionHover = (i: number, j: number) => {
  return actionHover.value === `${i},${j}`
}

const operationCellStyle = computed(() => {
  const shapeLayout = getShapeLayout(submitForm.value)
  return {
    width: `${cmTo600Dpi(shapeLayout.outer.width)}px`,
    height: `${cmTo600Dpi(shapeLayout.outer.height)}px`
  }
})

const operationButtonScaleStyle = computed(() => {
  return {
    '--operation-scale': btnScale.value,
    '--operation-offset': `${96 * btnScale.value}px`,
    '--column-operation-offset': `${38 * btnScale.value}px`
  }
})
</script>

<template>
  <main v-if="!authReady" class="auth-shell">
    <section class="auth-card">
      <p class="eyebrow">Loading</p>
      <h1>正在验证登录状态</h1>
      <p class="subcopy">请稍候。</p>
    </section>
  </main>
  <main v-else-if="!currentUser" class="auth-shell">
    <section class="auth-card">
      <p class="eyebrow">Account Login</p>
      <h1>吧唧打印图排版工具</h1>
      <p class="subcopy">请先登录账号，登录后才能使用打印图排版和导出工具。</p>
      <el-form class="auth-form" @submit.prevent="login">
        <label>
          账号
          <el-input v-model="loginForm.username" autocomplete="username"/>
        </label>
        <label>
          密码
          <el-input v-model="loginForm.password" type="password" autocomplete="current-password" show-password/>
        </label>
        <label class="remember-row">
          <el-checkbox v-model="loginForm.remember">记住密码</el-checkbox>
        </label>
        <el-button class="auth-submit" type="primary" native-type="submit" :loading="loginLoading">登录使用</el-button>
        <p class="auth-message">{{ loginMessage }}</p>
      </el-form>
    </section>
  </main>
  <div v-else class="index">
    <div ref="previewBox" class="preview-box">
      <div class="preview" ref="preview">
        <div class="column-actions" :style="operationButtonScaleStyle">
          <div
            class="column-action-cell"
            v-for="(_, j) in (images[0] || [])"
            :key="`column-action-${j}`"
            :style="operationCellStyle"
          >
            <div class="operation-buttons operation-buttons--column" :style="operationButtonScaleStyle">
              <el-tooltip content="把已复制图片粘贴到这一列" placement="top">
                <el-button :icon="CopyDocument" circle @click="fillColumnFromCopiedImage(j)"/>
              </el-tooltip>
              <el-tooltip content="清空这一列" placement="top">
                <el-button :icon="Delete" circle @click="clearColumn(j)"/>
              </el-tooltip>
            </div>
          </div>
        </div>
        <div class="corner-actions operation-buttons" :style="operationButtonScaleStyle">
          <el-tooltip content="把已复制图片粘贴到全部位置" placement="top">
            <el-button :icon="CopyDocument" circle @click="fillAllFromCopiedImage"/>
          </el-tooltip>
          <el-tooltip content="全部清空" placement="top">
            <el-button :icon="Delete" circle @click="clearAllImages"/>
          </el-tooltip>
        </div>
        <div class="row" v-for="(row, i) of images" :key="i">
          <div class="row-actions operation-buttons" :style="operationButtonScaleStyle">
            <el-tooltip content="把已复制图片粘贴到这一行" placement="left">
              <el-button :icon="CopyDocument" circle @click="fillRowFromCopiedImage(i)"/>
            </el-tooltip>
            <el-tooltip content="清空这一行" placement="left">
              <el-button :icon="Delete" circle @click="clearRow(i)"/>
            </el-tooltip>
          </div>
          <div
            ref="badgeBg"
            class="badge-bg"
            :class="[getShapeClass(submitForm.shape), {hidden: loading && !image}]"
            v-for="(image, j) in row"
            :key="j"
            @mouseenter="editable = `${i},${j}`"
            @mouseleave="editable = ''"
          >
            <div
              class="badge-bg-shape"
              :class="getShapeClass(submitForm.shape)"
              :style="getShapeStyle(submitForm)"
            ></div>
            <div
              ref="badge"
              class="badge"
              :class="getShapeClass(submitForm.shape)"
              :style="getShapeStyle(submitForm)"
            >
              <img v-if="image" class="image" :src="image" alt="" @click="addImage(i, j)"/>
            </div>
            <div
              class="badge-actions"
              :class="{'is-open': isEditableBadge(i, j)}"
              :style="badgeActionStyle"
              @mouseenter="actionHover = `${i},${j}`"
              @mouseleave="actionHover = ''"
            >
              <template v-if="!image">
                <el-tooltip content="添加图片" placement="top">
                  <el-button :icon="Plus" circle @click="addImage(i, j)"/>
                </el-tooltip>
                <el-tooltip v-if="isActionHover(i, j)" :content="copyItem ? '粘贴已复制图片' : '请先复制图片'" placement="top">
                  <el-button :icon="List" circle :disabled="!copyItem" @click="pasteImage(i, j)"/>
                </el-tooltip>
              </template>
              <template v-else-if="isEditableBadge(i, j)">
                <el-tooltip content="更换图片" placement="top">
                  <el-button :icon="Edit" circle @click="addImage(i, j)"/>
                </el-tooltip>
                <el-tooltip content="裁剪图片" placement="top">
                  <el-button :icon="Crop" circle @click="editImage(i, j)"/>
                </el-tooltip>
                <el-tooltip content="复制这张图片" placement="top">
                  <el-button :icon="CopyDocument" circle @click="copyImage(i, j)"/>
                </el-tooltip>
                <el-tooltip content="删除当前图片" placement="top">
                  <el-button :icon="Delete" circle @click="removeImage(i, j)"/>
                </el-tooltip>
              </template>
              <template v-else>
                <el-tooltip content="复制这张图片" placement="top">
                  <el-button :icon="CopyDocument" circle @click="copyImage(i, j)"/>
                </el-tooltip>
              </template>
            </div>
          </div>
        </div>
      </div>
    </div>
    <div class="content">
      <h1 class="title">吧唧打印图排版工具</h1>
      <div class="account-bar">
        <div>
          <span>当前账号</span>
          <strong>{{ currentUser.username }}</strong>
        </div>
        <div class="account-actions">
          <el-button v-if="currentUser.role === 'admin'" size="small" @click="toggleAdminPanel">
            员工账号管理
          </el-button>
          <el-button size="small" @click="logout">退出登录</el-button>
        </div>
      </div>
      <div v-if="currentUser.role === 'admin' && adminVisible" class="admin-panel">
        <div class="admin-panel__head">
          <div>
            <p class="eyebrow">Admin</p>
            <h2>员工账号管理</h2>
          </div>
          <p class="admin-message">{{ employeeMessage }}</p>
        </div>
        <div class="employee-form">
          <label>
            员工账号
            <el-input v-model="employeeForm.username" placeholder="输入员工账号"/>
          </label>
          <label>
            员工密码
            <el-input v-model="employeeForm.password" placeholder="输入员工密码"/>
          </label>
          <el-button @click="generateEmployeeCredentials">随机生成</el-button>
          <el-button type="primary" @click="createEmployeeAccount">新增员工</el-button>
        </div>
        <div class="employee-list">
          <p v-if="!employeeAccounts.length" class="muted">暂无员工账号</p>
          <div v-for="(account, index) in employeeAccounts" :key="account.id" class="employee-row">
            <span class="employee-index">{{ index + 1 }}</span>
            <label>
              账号
              <el-input :model-value="account.username" @input="(value: string) => updateEmployeeDraft(account.id, 'username', value)"/>
            </label>
            <label>
              新密码
              <el-input :model-value="account.password || ''" placeholder="留空则不修改密码" @input="(value: string) => updateEmployeeDraft(account.id, 'password', value)"/>
            </label>
            <el-button @click="updateEmployeeAccount(account, account.password || '')">保存</el-button>
            <el-button type="danger" plain @click="removeEmployeeAccount(account.id)">删除</el-button>
          </div>
        </div>
      </div>
      <div class="content-box">
        <el-form class="form" ref="ruleFormRef" :model="ruleForm" :rules="rules" label-position="left"
                 label-width="120px">
          <el-form-item label="纸张大小">
            <el-radio-group v-model="paper">
              <el-radio-button v-for="item in paperSize.entries()" :key="item[0]" :label="item[0]"></el-radio-button>
            </el-radio-group>
          </el-form-item>
          <el-form-item v-if="paper === '自定义'" label="纸张宽度(像素)" prop="width">
            <el-input-number v-model="ruleForm.width" controls-position="right" :min="0.01"/>
          </el-form-item>
          <el-form-item v-if="paper === '自定义'" label="纸张高度(像素)" prop="height">
            <el-input-number v-model="ruleForm.height" controls-position="right" :min="0.01"/>
          </el-form-item>
          <el-form-item label="形状" prop="shape">
            <el-select v-model="ruleForm.shape">
              <el-option v-for="item in shapeOptions" :key="item.value" :label="item.label" :value="item.value"/>
            </el-select>
          </el-form-item>
          <el-form-item label="行数" prop="row">
            <el-input-number v-model="ruleForm.row" controls-position="right" :min="1"/>
          </el-form-item>
          <el-form-item label="列数" prop="col">
            <el-input-number v-model="ruleForm.col" controls-position="right" :min="1"/>
          </el-form-item>
          <el-form-item v-if="ruleForm.shape === 'circle'" label="直径(cm)" prop="diam">
            <el-input-number v-model="ruleForm.diam" :precision="2" controls-position="right" :min="0.01"/>
          </el-form-item>
          <template v-if="ruleForm.shape === 'rectangle'">
            <el-form-item label="宽度(cm)" prop="rectWidth">
              <el-input-number v-model="ruleForm.rectWidth" :precision="2" controls-position="right" :min="0.01"/>
            </el-form-item>
            <el-form-item label="高度(cm)" prop="rectHeight">
              <el-input-number v-model="ruleForm.rectHeight" :precision="2" controls-position="right" :min="0.01"/>
            </el-form-item>
          </template>
          <template v-if="ruleForm.shape === 'ellipse'">
            <el-form-item label="长边(cm)" prop="ellipseLong">
              <el-input-number v-model="ruleForm.ellipseLong" :precision="2" controls-position="right" :min="0.01"/>
            </el-form-item>
            <el-form-item label="短边(cm)" prop="ellipseShort">
              <el-input-number v-model="ruleForm.ellipseShort" :precision="2" controls-position="right" :min="0.01"/>
            </el-form-item>
          </template>
          <template v-if="ruleForm.shape === 'polygon'">
            <el-form-item label="边数" prop="polygonSides">
              <el-input-number v-model="ruleForm.polygonSides" controls-position="right" :min="3" :step="1" :precision="0"/>
            </el-form-item>
            <el-form-item label="边长(cm)" prop="polygonSide">
              <el-input-number v-model="ruleForm.polygonSide" :precision="2" controls-position="right" :min="0.01"/>
            </el-form-item>
          </template>
          <el-form-item label="边距(cm)" prop="padding">
            <el-input-number v-model="ruleForm.padding" :precision="2" controls-position="right" :min="0"/>
          </el-form-item>
          <el-form-item label="边框颜色" prop="color">
            <el-color-picker v-model="ruleForm.color"/>
          </el-form-item>
        </el-form>
        <el-form class="form" ref="exportFormRef" :model="exportForm" label-position="left"
                 label-width="120px">
          <el-form-item label="文件名">
            <el-input v-model="exportForm.fileName"/>
          </el-form-item>
          <el-form-item label="文件类型">
            <el-select v-model="exportForm.fileType">
              <el-option label="png" value="png"/>
              <el-option label="jpeg" value="jpeg"/>
            </el-select>
          </el-form-item>
          <el-form-item label="质量">
            <el-slider v-model="exportForm.quality" :min="0.1" :max="1" :step="0.1"
                       :format-tooltip="(value: number) => value * 100 + '%'"/>
          </el-form-item>
          <el-form-item label="DPI">
            <el-select v-model="exportForm.dpi">
              <el-option label="600" :value="600"/>
              <el-option label="400" :value="400"/>
              <el-option label="300" :value="300"/>
            </el-select>
          </el-form-item>
        </el-form>
        <div class="btn-box">
          <el-button class="button" type="primary" @click="onSubmit">生成</el-button>
          <el-button :loading="loading" class="button" type="primary" @click="output">导出</el-button>
        </div>
        <p>Tips：当导出打印图时，未选图片的形状底框会被隐藏。</p>
      </div>
      <div class="footer">
        <img class="github-icon" src="../assets/github-mark.svg" @click="goGitHub" alt="GitHub"/>
      </div>
    </div>
    <el-dialog v-model="dialogVisible" title="裁剪" :close-on-click-modal="false">
      <cropper ref="cropperRef" :src="tempImages[rowIndex][colIndex]"></cropper>
      <template #footer>
        <span class="dialog-footer">
          <el-button @click="dialogVisible = false">取消</el-button>
          <el-button type="primary" @click="confirmCrop">确定</el-button>
        </span>
      </template>
    </el-dialog>
    <input ref="fileInput" type="file" class="file-input" accept="image/*" @change="fileChange"/>
  </div>
</template>

<style scoped>
.auth-shell {
  min-height: 100%;
  display: grid;
  place-items: center;
  padding: 24px;
  background:
    radial-gradient(circle at 18% 18%, rgba(232, 244, 239, 0.88), transparent 32%),
    linear-gradient(145deg, #f7f4ed 0%, #edf2ee 100%);
}

.auth-card {
  width: min(520px, 100%);
  border: 1px solid #d9ddd7;
  border-radius: 8px;
  padding: 32px;
  background: rgba(255, 255, 255, 0.86);
  box-shadow: 0 22px 60px rgba(24, 34, 29, 0.12);

  h1 {
    margin: 0 0 12px;
    font-size: 28px;
    line-height: 1.2;
  }
}

.eyebrow {
  margin: 0 0 10px;
  color: #5f6d66;
  font-size: 12px;
  font-weight: 700;
  letter-spacing: 0;
  text-transform: uppercase;
}

.subcopy {
  margin: 0 0 22px;
  color: #56615b;
  line-height: 1.6;
}

.auth-form {
  display: grid;
  gap: 14px;

  label {
    display: grid;
    gap: 8px;
    color: #303133;
    font-size: 14px;
    font-weight: 700;
  }

  .remember-row {
    display: flex;
    align-items: center;
    gap: 8px;
    margin-top: -4px;
  }
}

.auth-submit {
  margin-top: 4px;
}

.auth-message,
.admin-message {
  min-height: 20px;
  margin: 0;
  color: #b42318;
  font-size: 13px;
  font-weight: 700;
}

.muted {
  margin: 0;
  color: #909399;
  font-size: 13px;
}

.index {
  display: flex;
  width: 100%;
  height: 100%;
  padding: 16px;
  gap: 12px;
  background: #f6f4ef;

  .file-input {
    width: 0;
    height: 0;
    opacity: 0;
  }

  .preview-box {
    flex: 3;
    display: flex;
    align-items: center;
    justify-content: center;
    border: 1px solid #d9ddd7;
    border-radius: 8px;
    background:
      linear-gradient(45deg, rgba(221, 221, 221, 0.56) 25%, transparent 25%),
      linear-gradient(-45deg, rgba(221, 221, 221, 0.56) 25%, transparent 25%),
      linear-gradient(45deg, transparent 75%, rgba(221, 221, 221, 0.56) 75%),
      linear-gradient(-45deg, transparent 75%, rgba(221, 221, 221, 0.56) 75%),
      #f4f5f2;
    background-position:
      0 0,
      0 10px,
      10px -10px,
      -10px 0;
    background-size:
      20px 20px,
      20px 20px,
      20px 20px,
      20px 20px,
      auto;
    height: 100%;
    overflow: hidden;
    box-shadow: 0 18px 42px rgba(35, 42, 38, 0.13);

    .preview {
      position: relative;
      width: 100%;
      height: 100%;
      background: #fff;
      display: flex;
      align-items: center;
      justify-content: space-evenly;
      flex-direction: column;
      overflow: visible;

      .column-actions {
        position: absolute;
        top: calc(var(--column-operation-offset) * -1);
        left: 0;
        width: 100%;
        display: flex;
        align-items: center;
        justify-content: space-evenly;
        pointer-events: none;
      }

      .column-action-cell {
        display: flex;
        align-items: center;
        justify-content: center;
        pointer-events: auto;
      }

      .corner-actions {
        position: absolute;
        top: calc(var(--operation-offset) * -1 + 14px);
        left: calc(var(--operation-offset) * -1 - 18px);
        z-index: 30;
        pointer-events: auto;
        transform: scale(var(--operation-scale));
        transform-origin: top left;
      }

      .operation-buttons {
        display: flex;
        align-items: center;
        justify-content: center;
        gap: 3px;
        padding: 2px;
        border-radius: 999px;
        background: rgba(255, 255, 255, 0.96);
        border: 1px solid rgba(42, 59, 53, 0.12);
        box-shadow: 0 8px 22px rgba(29, 44, 39, 0.14);
        transform: scale(var(--operation-scale));
        transform-origin: center;

        :deep(.el-button) {
          width: 16px;
          height: 16px;
          min-height: 16px;
          margin: 0;
          color: #32413b;
          background: #fff;
          border-color: rgba(40, 58, 52, 0.14);
        }

        :deep(.el-button:hover) {
          color: #1f5e52;
          border-color: #1f5e52;
        }
      }

      .row {
        position: relative;
        width: 100%;
        display: flex;
        align-items: center;
        justify-content: space-evenly;

        .row-actions {
          position: absolute;
          left: calc(var(--operation-offset) * -1);
          top: 50%;
          z-index: 30;
          pointer-events: auto;
          transform: translateY(-50%) scale(var(--operation-scale));
          transform-origin: center left;

          :deep(.el-button) {
            width: 16px;
            height: 16px;
            min-height: 16px;
            margin: 0;
            color: #32413b;
            background: #fff;
            border-color: rgba(40, 58, 52, 0.14);
          }

          :deep(.el-button:hover) {
            color: #1f5e52;
            border-color: #1f5e52;
          }
        }

        .badge-bg {
          position: relative;
          background: transparent;
          display: flex;
          align-items: center;
          justify-content: center;
          overflow: visible;

          &.hidden {
            opacity: 0;
          }

          .badge-bg-shape {
            position: absolute;
            inset: 0;
            background: var(--badge-fill-color, rgba(0, 0, 0, 0.1));
            pointer-events: none;
          }

          .badge {
            position: relative;
            overflow: hidden;
            z-index: 1;

            .image {
              width: 100%;
              height: 100%;
              object-fit: cover;
              cursor: pointer;
            }
          }

          .badge-actions {
            position: absolute;
            top: 50%;
            left: 50%;
            z-index: 25;
            display: grid;
            grid-template-columns: repeat(2, max-content);
            grid-auto-rows: max-content;
            align-items: center;
            justify-content: center;
            gap: 4px;
            transform: translate(-50%, -50%) scale(var(--badge-action-scale));
            pointer-events: auto;
            transform-origin: center;

            :deep(.el-button) {
              width: 20px;
              height: 20px;
              min-height: 20px;
              margin: 0;
              color: #606b66;
              background: rgba(255, 255, 255, 0.96);
              border-color: rgba(70, 82, 77, 0.14);
              box-shadow: 0 4px 12px rgba(28, 39, 35, 0.14);
              transition: transform 120ms ease, box-shadow 120ms ease, border-color 120ms ease;
            }

            :deep(.el-button:hover) {
              border-color: rgba(31, 94, 82, 0.36);
              box-shadow: 0 6px 14px rgba(28, 39, 35, 0.18);
            }

            :deep(.el-button:active) {
              transform: scale(0.88);
              box-shadow: 0 2px 7px rgba(28, 39, 35, 0.18);
            }

          }
        }
      }
    }
  }

  .content {
    flex: 1;
    width: 100%;
    min-width: 360px;
    max-width: 520px;
    padding: 18px;
    display: flex;
    flex-direction: column;
    align-items: center;
    overflow-y: auto;
    border: 1px solid #d9ddd7;
    border-radius: 8px;
    background: rgba(255, 255, 255, 0.82);
    box-shadow: 0 18px 42px rgba(35, 42, 38, 0.1);

    .content-box {
      flex: 1;
      width: 100%;
    }

    .title {
      font-size: 24px;
      font-weight: 900;
      width: 100%;
      text-align: center;
      margin: 0;
      color: #151816;
      letter-spacing: 0;
    }

    .account-bar {
      width: 100%;
      display: flex;
      align-items: center;
      justify-content: space-between;
      gap: 12px;
      border: 1px solid #dcdfe6;
      border-radius: 8px;
      margin-top: 16px;
      padding: 10px 12px;
      background: #f7faf8;

      span {
        display: block;
        color: #909399;
        font-size: 12px;
      }

      strong {
        color: #303133;
        font-size: 14px;
      }
    }

    .account-actions {
      display: flex;
      gap: 8px;
      flex-wrap: wrap;
      justify-content: flex-end;
    }

    .admin-panel {
      width: 100%;
      border: 1px solid #d9ddd7;
      border-radius: 8px;
      margin-top: 12px;
      padding: 14px;
      background: #fff;
    }

    .admin-panel__head {
      display: flex;
      align-items: flex-start;
      justify-content: space-between;
      gap: 12px;
      margin-bottom: 12px;

      h2 {
        margin: 0;
        font-size: 18px;
      }
    }

    .employee-form,
    .employee-list,
    .employee-row {
      display: grid;
      gap: 10px;
    }

    .employee-form {
      grid-template-columns: minmax(0, 1fr);
      border-bottom: 1px solid #ebeef5;
      padding-bottom: 12px;
      margin-bottom: 12px;

      label {
        display: grid;
        gap: 6px;
        color: #303133;
        font-size: 13px;
        font-weight: 700;
      }
    }

    .employee-row {
      grid-template-columns: 28px minmax(0, 1fr);
      border: 1px solid #ebeef5;
      border-radius: 8px;
      padding: 10px;

      label {
        display: grid;
        gap: 6px;
        color: #303133;
        font-size: 13px;
        font-weight: 700;
      }
    }

    .employee-index {
      width: 24px;
      height: 24px;
      display: grid;
      place-items: center;
      border-radius: 999px;
      color: #409eff;
      background: #ecf5ff;
      font-size: 12px;
      font-weight: 800;
      align-self: center;
    }

    .form {
      border-top: 1px solid #dfe3de;
      margin-top: 24px;
      padding-top: 22px;
    }

    .btn-box {
      display: flex;
      justify-content: space-evenly;
      align-items: center;
      width: 100%;
      margin-top: 40px;
      margin-bottom: 20px;

      .button {
        width: 100px;
        min-height: 42px;
        border: 0;
        font-weight: 800;
        background: #1f5e52;
        box-shadow: 0 14px 30px rgba(31, 94, 82, 0.22);

        &:first-child {
          margin-right: 10px;
        }
      }
    }

    .footer {
      display: flex;
      align-items: center;
      justify-content: flex-end;
      width: 100%;

      .github-icon {
        width: 1em;
        height: 1em;
        cursor: pointer;
        margin-left: 10px;
      }
    }
  }
}

.shape-circle {
  border-radius: 50%;
}

.shape-rectangle {
  border-radius: 0;
}

.shape-ellipse {
  border-radius: 50%;
}

:deep(.el-button--primary) {
  --el-button-bg-color: #1f5e52;
  --el-button-border-color: #1f5e52;
  --el-button-hover-bg-color: #2d7366;
  --el-button-hover-border-color: #2d7366;
  --el-button-active-bg-color: #17483f;
  --el-button-active-border-color: #17483f;
  font-weight: 800;
}

:deep(.el-radio-button__original-radio:checked + .el-radio-button__inner) {
  background-color: #1f5e52;
  border-color: #1f5e52;
  box-shadow: -1px 0 0 0 #1f5e52;
}

:deep(.el-form-item__label) {
  color: #3e4842;
  font-weight: 800;
}

:deep(.el-input__wrapper),
:deep(.el-input-number__decrease),
:deep(.el-input-number__increase),
:deep(.el-select__wrapper) {
  border-radius: 8px;
}

:deep(.el-slider__bar) {
  background-color: #1f5e52;
}

:deep(.el-slider__button) {
  border-color: #1f5e52;
}

</style>
