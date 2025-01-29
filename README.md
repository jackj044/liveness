# liveness


import UIKit
import ARKit

class ViewController: UIViewController, ARSCNViewDelegate, UIGestureRecognizerDelegate {
    
    @IBOutlet var sceneView: ARSCNView!
    
    private var actionVerified = false
    private var challengeStartTime: Date?
    
    private var blinkDetected = false
    private var smileDetected = false
    private var headTurnDetected = false
    
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
    
    private func startNewChallenge() {
        actionVerified = false
        challengeStartTime = Date()
        faceMotionHistory.removeAll() // Reset motion history
        
        blinkDetected = false
        smileDetected = false
        headTurnDetected = false
        
        print("🔍 Liveness Challenge: You must Smile & Blink. Head turn is optional.")
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
        
        // Liveness Challenge Detection
        if !actionVerified {
            checkLiveness(faceAnchor)
        }
    }
    
    private func checkLiveness(_ faceAnchor: ARFaceAnchor) {
        let blendShapes = faceAnchor.blendShapes
        
        // Detect Blink
        if let leftBlink = blendShapes[.eyeBlinkLeft]?.floatValue,
           let rightBlink = blendShapes[.eyeBlinkRight]?.floatValue,
           leftBlink > 0.7 && rightBlink > 0.7 {
            blinkDetected = true
            print("✅ Blink detected")
        }
        
        // Detect Smile
        if let mouthSmile = blendShapes[.mouthSmileLeft]?.floatValue,
           mouthSmile > 0.5 {
            smileDetected = true
            print("✅ Smile detected")
        }
        
        // Detect Optional Head Turn (Left or Right)
        if faceAnchor.transform.columns.3.x < -0.2 || faceAnchor.transform.columns.3.x > 0.2 {
            headTurnDetected = true
            print("✅ Head Turn detected (optional)")
        }
        
        // Verify liveness if both mandatory actions are completed
        if blinkDetected && smileDetected {
            verifyLiveness()
        }
    }
    
    private func verifyLiveness() {
        actionVerified = true
        let timeTaken = Date().timeIntervalSince(challengeStartTime ?? Date())
        
        if timeTaken > 5.0 {
            print("⚠️ Liveness challenge took too long. Possible spoofing attempt!")
            startNewChallenge()
        } else {
            print("✅ Liveness Verified: Smile & Blink detected. Head turn was \(headTurnDetected ? "also detected (extra verification)" : "not required").")
        }
    }
    
    // MARK: - Face Occlusion Detection
    private func isFaceOccluded(_ faceAnchor: ARFaceAnchor) -> Bool {
        let blendShapes = faceAnchor.blendShapes
        
        // Check occlusion levels for eyes, mouth, and cheeks
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
            
            // Store movement history
            faceMotionHistory.append(movement)
            if faceMotionHistory.count > 10 {
                faceMotionHistory.removeFirst()
            }
            
            // Analyze movement for natural variation
            let movementVariance = faceMotionHistory.reduce(0, +) / Float(faceMotionHistory.count)
            if movementVariance < 0.0001 { // Almost no movement means static image
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
        
        // Default position (closer to the face)
        cameraNode.position = SCNVector3(0, 0, 0.3) // Adjust Z to zoom
        
        // Default Field of View
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
        
        // Zoom by moving the camera closer/farther
        let newZ = max(0.1, min(0.6, cameraNode.position.z - (scale - 1) * 0.05)) // Keep within range
        cameraNode.position.z = newZ
        
        // Adjust FOV (lower value = zoom in)
        let newFOV = max(30, min(70, camera.fieldOfView - Double((scale - 1) * 5)))
        camera.fieldOfView = newFOV
        
        gesture.scale = 1.0 // Reset scale after applying zoom
    }
}
