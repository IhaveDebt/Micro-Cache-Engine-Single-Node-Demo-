import Foundation
import CryptoKit

// MARK: - Distributed Cache Node (Single Node Simulation)
class CacheItem: Codable {
    let key: String
    let value: Data
    let timestamp: Date
    let ttl: TimeInterval
    
    var isExpired: Bool {
        return Date().timeIntervalSince(timestamp) > ttl
    }
}

class DistributedCacheNode {
    private var storage: [String: CacheItem] = [:]
    private let queue = DispatchQueue(label: "cache.node.queue", qos: .userInitiated)
    
    func set(key: String, value: String, ttl: TimeInterval = 300) {
        queue.sync {
            let item = CacheItem(
                key: key,
                value: value.data(using: .utf8)!,
                timestamp: Date(),
                ttl: ttl
            )
            storage[key] = item
        }
    }
    
    func get(key: String) -> String? {
        return queue.sync {
            guard let item = storage[key] else { return nil }
            if item.isExpired {
                storage.removeValue(forKey: key)
                return nil
            }
            return String(decoding: item.value, as: UTF8.self)
        }
    }
    
    func evictExpired() {
        queue.sync {
            storage = storage.filter { !$0.value.isExpired }
        }
    }
    
    func nodeFingerprint() -> String {
        let encoder = JSONEncoder()
        let json = try! encoder.encode(storage)
        let hash = SHA256.hash(data: json)
        return hash.map { String(format: "%02x", $0) }.joined()
    }
}

// MARK: - Demo
let node = DistributedCacheNode()
node.set(key: "session_1", value: "UserA", ttl: 5)
node.set(key: "session_2", value: "UserB", ttl: 30)

print("[Node Boot] Fingerprint:", node.nodeFingerprint())

sleep(6)
node.evictExpired()

print("session_1:", node.get(key: "session_1") ?? "<expired>")
print("session_2:", node.get(key: "session_2") ?? "<present>")

print("[Node After Eviction] Fingerprint:", node.nodeFingerprint())

