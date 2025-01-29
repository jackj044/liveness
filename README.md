# liveness

import UIKit
import ARKit

class ViewController: UIViewController, ARSCNViewDelegate {
    
    @IBOutlet var sceneView: ARSCNView!
    private var requiredAction: LivenessAction = .blink
    private var actionVerified = false
    private var challengeStartTime: Date?
    
    private var previousFacePosition: SIMD3<Float>?
    private var faceMotionHistory: [Float] = []
    
    override func viewDidLoad() {
        super.viewDidLoad()
        
        guard ARFaceTrackingConfiguration.isSupported else {
            fatalError("Face tracking is not supported on this device.")
        }
        
        sceneView.delegate = self
        sceneView.session.run(ARFaceTrackingConfiguration())
        
        startNewChallenge()
    }
    
    override func viewWillDisappear(_ animated: Bool) {
        super.viewWillDisappear(animated)
        sceneView.session.pause()
    }
    
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
        requiredAction = LivenessAction.allCases.randomElement() ?? .blink
        actionVerified = false
        challengeStartTime = Date()
        faceMotionHistory.removeAll() // Reset motion history
        
        print("🔍 Liveness Challenge: \(requiredAction.description)")
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
        
        switch requiredAction {
        case .blink:
            if let leftBlink = blendShapes[.eyeBlinkLeft]?.floatValue,
               let rightBlink = blendShapes[.eyeBlinkRight]?.floatValue,
               leftBlink > 0.7 && rightBlink > 0.7 {
                verifyLiveness()
            }
            
        case .headTurnLeft:
            if faceAnchor.transform.columns.3.x < -0.2 {
                verifyLiveness()
            }
            
        case .headTurnRight:
            if faceAnchor.transform.columns.3.x > 0.2 {
                verifyLiveness()
            }
            
        case .smile:
            if let mouthSmile = blendShapes[.mouthSmileLeft]?.floatValue,
               mouthSmile > 0.5 {
                verifyLiveness()
            }
        }
    }
    
    private func verifyLiveness() {
        actionVerified = true
        let timeTaken = Date().timeIntervalSince(challengeStartTime ?? Date())
        
        if timeTaken > 5.0 {
            print("⚠️ Liveness challenge took too long. Possible spoofing attempt!")
            startNewChallenge()
        } else {
            print("✅ Liveness Verified: \(requiredAction.description)")
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
}
