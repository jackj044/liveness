# liveness

import UIKit
import ARKit
import Vision

class FaceDetectionViewController: UIViewController, ARSCNViewDelegate, ARSessionDelegate {
    
    let sceneView = ARSCNView()
    let overlayImageView = UIImageView()
    
    override func viewDidLoad() {
        super.viewDidLoad()
        
        setupARScene()
        setupOverlay()
    }
    
    func setupARScene() {
        sceneView.frame = view.bounds
        sceneView.delegate = self
        sceneView.session.delegate = self
        view.addSubview(sceneView)
        
        let configuration = ARFaceTrackingConfiguration()
        sceneView.session.run(configuration)
    }
    
    func setupOverlay() {
        overlayImageView.frame = view.bounds
        overlayImageView.contentMode = .scaleAspectFit
        overlayImageView.image = createOverlayImage()
        view.addSubview(overlayImageView)
    }
    
    func createOverlayImage() -> UIImage {
        let size = view.bounds.size
        UIGraphicsBeginImageContextWithOptions(size, false, 0)
        guard let context = UIGraphicsGetCurrentContext() else { return UIImage() }
        
        context.setFillColor(UIColor.black.withAlphaComponent(0.6).cgColor)
        context.fill(CGRect(origin: .zero, size: size))
        
        let circleSize: CGFloat = 200
        let circleRect = CGRect(x: (size.width - circleSize) / 2, y: (size.height - circleSize) / 2, width: circleSize, height: circleSize)
        
        context.setBlendMode(.clear)
        context.fillEllipse(in: circleRect)
        
        let image = UIGraphicsGetImageFromCurrentImageContext()
        UIGraphicsEndImageContext()
        
        return image ?? UIImage()
    }
    
    func session(_ session: ARSession, didUpdate anchors: [ARAnchor]) {
        guard let faceAnchor = anchors.first as? ARFaceAnchor else { return }
        
        DispatchQueue.main.async {
            self.processFaceAnchor(faceAnchor)
        }
    }
    
    func processFaceAnchor(_ faceAnchor: ARFaceAnchor) {
        let facePosition = sceneView.projectPoint(SCNVector3(faceAnchor.transform.columns.3.x, faceAnchor.transform.columns.3.y, faceAnchor.transform.columns.3.z))
        
        let facePoint = CGPoint(x: CGFloat(facePosition.x), y: CGFloat(facePosition.y))
        let circleRect = CGRect(x: (view.bounds.width - 200) / 2, y: (view.bounds.height - 200) / 2, width: 200, height: 200)
        
        if circleRect.contains(facePoint) {
            print("✅ Face is inside the circle!")
        } else {
            print("❌ Face is outside the circle")
        }
    }
}




import UIKit
import ARKit

class ViewController: UIViewController, ARSCNViewDelegate, UIGestureRecognizerDelegate {
    
    @IBOutlet var sceneView: ARSCNView!
    
    private var requiredActions: [LivenessAction] = [.blink, .smile] // Mandatory actions
    private var optionalActions: [LivenessAction] = [.headTurnLeft, .headTurnRight] // Optional actions
    
    private var completedActions: Set<LivenessAction> = []
    private var challengeStartTime: Date?
    
    private var previousFacePosition: SIMD3<Float>?
    private var faceMotionHistory: [Float] = []
    
    private var cameraNode: SCNNode! // Camera node for zooming
    
    override func viewDidLoad() {
        super.viewDidLoad()
        
        guard ARFaceTrackingConfiguration.isSupported else {
            fatalError("Face tracking is not supported on this device.")
        }
        
        sceneView.delegate = self
        sceneView.session.run(ARFaceTrackingConfiguration())
        
        setupCameraNode()
        setupPinchToZoomGesture()
        startNewChallenge()
    }
    
    override func viewWillDisappear(_ animated: Bool) {
        super.viewWillDisappear(animated)
        sceneView.session.pause()
    }
    
    // MARK: - Liveness Actions Enum
    enum LivenessAction: CaseIterable {
        case blink, headTurnLeft, headTurnRight, smile
        
        var description: String {
            switch self {
            case .blink: return "Blink your eyes"
            case .headTurnLeft: return "Turn your head left"
            case .headTurnRight: return "Turn your head right"
            case .smile: return "Smile"
            }
        }
    }
    
    private func startNewChallenge() {
        completedActions.removeAll()
        challengeStartTime = Date()
        faceMotionHistory.removeAll()
        
        print("🔍 Liveness Challenge: Perform all required actions: \(requiredActions.map { $0.description }.joined(separator: ", "))")
    }
    
    func renderer(_ renderer: SCNSceneRenderer, nodeFor anchor: ARAnchor) -> SCNNode? {
        guard let faceAnchor = anchor as? ARFaceAnchor else { return nil }
        
        let faceNode = SCNNode()
        let faceGeometry = ARSCNFaceGeometry(device: sceneView.device!)
        faceNode.geometry = faceGeometry
        return faceNode
    }
    
    func renderer(_ renderer: SCNSceneRenderer, didUpdate node: SCNNode, for anchor: ARAnchor) {
        guard let faceAnchor = anchor as? ARFaceAnchor else { return }
        
        // Check face occlusion before processing liveness
        if isFaceOccluded(faceAnchor) {
            print("⚠️ Face is partially or fully occluded! Possible spoof attempt.")
            return
        }
        
        // Natural motion check
        if !isFaceMovingNaturally(faceAnchor) {
            print("⚠️ Possible spoofing detected: Face motion is unnatural!")
            return
        }
        
        // Check liveness dynamically
        checkLiveness(faceAnchor)
    }
    
    private func checkLiveness(_ faceAnchor: ARFaceAnchor) {
        let blendShapes = faceAnchor.blendShapes
        
        for action in LivenessAction.allCases {
            switch action {
            case .blink:
                if let leftBlink = blendShapes[.eyeBlinkLeft]?.floatValue,
                   let rightBlink = blendShapes[.eyeBlinkRight]?.floatValue,
                   leftBlink > 0.7 && rightBlink > 0.7 {
                    markActionCompleted(action)
                }
                
            case .headTurnLeft:
                if faceAnchor.transform.columns.3.x < -0.2 {
                    markActionCompleted(action)
                }
                
            case .headTurnRight:
                if faceAnchor.transform.columns.3.x > 0.2 {
                    markActionCompleted(action)
                }
                
            case .smile:
                if let mouthSmile = blendShapes[.mouthSmileLeft]?.floatValue,
                   mouthSmile > 0.5 {
                    markActionCompleted(action)
                }
            }
        }
        
        verifyLiveness()
    }
    
    private func markActionCompleted(_ action: LivenessAction) {
        if !completedActions.contains(action) {
            completedActions.insert(action)
            print("✅ Action Completed: \(action.description)")
        }
    }
    
    private func verifyLiveness() {
        let timeTaken = Date().timeIntervalSince(challengeStartTime ?? Date())
        
        let allMandatoryCompleted = requiredActions.allSatisfy { completedActions.contains($0) }
        
        if allMandatoryCompleted {
            print("✅ Liveness Verified: All mandatory actions completed.")
            if timeTaken > 5.0 {
                print("⚠️ Liveness challenge took too long. Possible spoofing attempt!")
            }
        }
    }
    
    // MARK: - Face Occlusion Detection
    private func isFaceOccluded(_ faceAnchor: ARFaceAnchor) -> Bool {
        let blendShapes = faceAnchor.blendShapes
        
        if let eyeLeftClosed = blendShapes[.eyeBlinkLeft]?.floatValue,
           let eyeRightClosed = blendShapes[.eyeBlinkRight]?.floatValue,
           let mouthClose = blendShapes[.jawOpen]?.floatValue,
           let cheekPuff = blendShapes[.cheekPuff]?.floatValue {
            
            if eyeLeftClosed < 0.1 && eyeRightClosed < 0.1 {
                print("⚠️ Eyes may be occluded or covered!")
                return true
            }
            
            if mouthClose < 0.1 {
                print("⚠️ Mouth may be occluded!")
                return true
            }
            
            if cheekPuff > 0.5 {
                print("⚠️ Unusual cheek detection (possible mask/spoofing)!")
                return true
            }
        }
        
        return false
    }
    
    // MARK: - Natural Motion Check
    private func isFaceMovingNaturally(_ faceAnchor: ARFaceAnchor) -> Bool {
        let currentFacePosition = faceAnchor.transform.columns.3
        
        if let previousPosition = previousFacePosition {
            let movement = simd_distance(previousPosition, currentFacePosition)
            
            faceMotionHistory.append(movement)
            if faceMotionHistory.count > 10 {
                faceMotionHistory.removeFirst()
            }
            
            let movementVariance = faceMotionHistory.reduce(0, +) / Float(faceMotionHistory.count)
            if movementVariance < 0.0001 {
                return false
            }
        }
        
        previousFacePosition = currentFacePosition
        return true
    }
    
    // MARK: - Zoom Functions
    private func setupCameraNode() {
        cameraNode = SCNNode()
        cameraNode.camera = SCNCamera()
        
        cameraNode.position = SCNVector3(0, 0, 0.3)
        cameraNode.camera?.fieldOfView = 50
        
        sceneView.pointOfView = cameraNode
        sceneView.scene.rootNode.addChildNode(cameraNode)
    }
    
    private func setupPinchToZoomGesture() {
        let pinchGesture = UIPinchGestureRecognizer(target: self, action: #selector(handlePinchGesture(_:)))
        sceneView.addGestureRecognizer(pinchGesture)
    }
    
    @objc private func handlePinchGesture(_ gesture: UIPinchGestureRecognizer) {
        guard let camera = cameraNode.camera else { return }
        
        let scale = Float(gesture.scale)
        
        let newZ = max(0.1, min(0.6, cameraNode.position.z - (scale - 1) * 0.05))
        cameraNode.position.z = newZ
        
        let newFOV = max(30, min(70, camera.fieldOfView - Double((scale - 1) * 5)))
        camera.fieldOfView = newFOV
        
        gesture.scale = 1.0
    }
}
